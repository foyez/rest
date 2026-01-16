# 7.2 API Documentation

## Why Document APIs?

**Without documentation:**
```
Developer: "How do I use this API?"
You: "Read the code" 🤷
Developer: *gives up and uses competitor's API*
```

**With documentation:**
```
Developer: "How do I use this API?"
Documentation: "Here's everything you need" 📚
Developer: *successfully integrates in 30 minutes*
```

---

## What Makes Good Documentation?

✅ **Good documentation:**
- Getting started guide
- Complete endpoint reference
- Code examples
- Error codes explained
- Authentication instructions
- Interactive playground
- Search functionality

❌ **Bad documentation:**
- Just a list of endpoints
- No examples
- Outdated
- Missing error codes
- Hard to navigate

---

## Documentation Standards

### 1. OpenAPI (Swagger)

**Industry standard for REST APIs**

```yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
  description: API for managing users

servers:
  - url: https://api.example.com/v1

paths:
  /users:
    get:
      summary: List all users
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 10
      responses:
        '200':
          description: Successful response
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/User'
    
    post:
      summary: Create a user
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateUserRequest'
      responses:
        '201':
          description: User created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '400':
          description: Invalid input

  /users/{id}:
    get:
      summary: Get user by ID
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: User found
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
        '404':
          description: User not found

components:
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          example: 123
        name:
          type: string
          example: John Doe
        email:
          type: string
          format: email
          example: john@example.com
        created_at:
          type: string
          format: date-time
          example: "2025-01-16T10:00:00Z"
    
    CreateUserRequest:
      type: object
      required:
        - name
        - email
      properties:
        name:
          type: string
          minLength: 3
          maxLength: 50
        email:
          type: string
          format: email

  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT

security:
  - bearerAuth: []
```

---

### 2. API Blueprint

**Markdown-based documentation**

```markdown
# User API

## Users Collection [/users]

### List Users [GET]

+ Parameters
    + page: 1 (number, optional) - Page number
    + limit: 10 (number, optional) - Items per page

+ Response 200 (application/json)
    + Attributes (array[User])

### Create User [POST]

+ Request (application/json)
    + Attributes
        + name: John Doe (string, required)
        + email: john@example.com (string, required)

+ Response 201 (application/json)
    + Attributes (User)

## User [/users/{id}]

+ Parameters
    + id: 123 (number) - User ID

### Get User [GET]

+ Response 200 (application/json)
    + Attributes (User)

+ Response 404 (application/json)
    + Attributes
        + error: User not found

# Data Structures

## User
+ id: 123 (number)
+ name: John Doe (string)
+ email: john@example.com (string)
+ created_at: 2025-01-16T10:00:00Z (string)
```

---

## Generating Documentation

### From Code Comments (Go)

```go
package main

// User represents a user in the system
type User struct {
    ID        int       `json:"id" example:"123"`
    Name      string    `json:"name" example:"John Doe"`
    Email     string    `json:"email" example:"john@example.com"`
    CreatedAt time.Time `json:"created_at" example:"2025-01-16T10:00:00Z"`
}

// CreateUserRequest is the request body for creating a user
type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=3,max=50" example:"John Doe"`
    Email string `json:"email" binding:"required,email" example:"john@example.com"`
}

// @Summary List all users
// @Description Get a paginated list of users
// @Tags users
// @Accept json
// @Produce json
// @Param page query int false "Page number" default(1)
// @Param limit query int false "Items per page" default(10)
// @Success 200 {array} User
// @Router /users [get]
func GetUsers(c *gin.Context) {
    // Implementation...
}

// @Summary Create a user
// @Description Create a new user
// @Tags users
// @Accept json
// @Produce json
// @Param user body CreateUserRequest true "User data"
// @Success 201 {object} User
// @Failure 400 {object} ErrorResponse
// @Router /users [post]
func CreateUser(c *gin.Context) {
    // Implementation...
}

// @Summary Get user by ID
// @Description Get a single user by ID
// @Tags users
// @Accept json
// @Produce json
// @Param id path int true "User ID"
// @Success 200 {object} User
// @Failure 404 {object} ErrorResponse
// @Router /users/{id} [get]
func GetUser(c *gin.Context) {
    // Implementation...
}
```

**Generate with Swaggo:**

```bash
# Install
go install github.com/swaggo/swag/cmd/swag@latest

# Generate docs
swag init

# This creates:
# - docs/docs.go
# - docs/swagger.json
# - docs/swagger.yaml
```

**Serve Swagger UI:**

```go
import (
    "github.com/gin-gonic/gin"
    swaggerFiles "github.com/swaggo/files"
    ginSwagger "github.com/swaggo/gin-swagger"
)

func main() {
    r := gin.Default()
    
    // API routes
    r.GET("/users", GetUsers)
    r.POST("/users", CreateUser)
    r.GET("/users/:id", GetUser)
    
    // Swagger UI
    r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))
    
    r.Run(":8080")
}

// Access at: http://localhost:8080/swagger/index.html
```

---

## Documentation Sections

### 1. Getting Started

```markdown
# Getting Started

## Base URL
```
https://api.example.com/v1
```

## Authentication
All requests require an API key in the header:
```
Authorization: Bearer YOUR_API_KEY
```

## Quick Example
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.example.com/v1/users
```

## Rate Limits
- 100 requests per minute
- 5000 requests per day
```

---

### 2. Authentication

```markdown
# Authentication

## Getting an API Key
1. Sign up at https://example.com/signup
2. Go to Dashboard → API Keys
3. Click "Create API Key"
4. Copy your key (shown only once!)

## Using the API Key

### Header (Recommended)
```bash
curl -H "Authorization: Bearer sk_live_abc123..." \
  https://api.example.com/v1/users
```

### Query Parameter (Not Recommended)
```bash
curl "https://api.example.com/v1/users?api_key=sk_live_abc123..."
```

## Token Types
- `sk_test_*` - Test mode (sandbox)
- `sk_live_*` - Production mode
```

---

### 3. Endpoints Reference

```markdown
# Endpoints

## List Users
`GET /users`

Returns a paginated list of users.

### Query Parameters
| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| page | integer | No | 1 | Page number |
| limit | integer | No | 10 | Items per page |
| sort | string | No | created_at | Sort field |
| order | string | No | desc | Sort order (asc/desc) |

### Response
```json
{
  "data": [
    {
      "id": 123,
      "name": "John Doe",
      "email": "john@example.com",
      "created_at": "2025-01-16T10:00:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "total_pages": 10
  }
}
```

### Example
```bash
curl -H "Authorization: Bearer YOUR_API_KEY" \
  "https://api.example.com/v1/users?page=1&limit=20"
```

---

## Create User
`POST /users`

Creates a new user.

### Request Body
```json
{
  "name": "John Doe",
  "email": "john@example.com"
}
```

### Response `201 Created`
```json
{
  "id": 123,
  "name": "John Doe",
  "email": "john@example.com",
  "created_at": "2025-01-16T10:00:00Z"
}
```

### Errors
- `400 Bad Request` - Invalid input
- `409 Conflict` - Email already exists

### Example
```bash
curl -X POST \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}' \
  https://api.example.com/v1/users
```
```

---

### 4. Error Codes

```markdown
# Error Codes

All errors follow this format:
```json
{
  "error": {
    "code": "validation_error",
    "message": "Invalid email format",
    "details": {
      "field": "email",
      "value": "invalid-email"
    }
  }
}
```

## HTTP Status Codes
| Code | Meaning | Description |
|------|---------|-------------|
| 200 | OK | Request succeeded |
| 201 | Created | Resource created |
| 204 | No Content | Successful, no response body |
| 400 | Bad Request | Invalid input |
| 401 | Unauthorized | Missing or invalid API key |
| 403 | Forbidden | Insufficient permissions |
| 404 | Not Found | Resource doesn't exist |
| 409 | Conflict | Resource already exists |
| 422 | Unprocessable Entity | Validation failed |
| 429 | Too Many Requests | Rate limit exceeded |
| 500 | Internal Server Error | Server error |

## Error Codes
| Code | Description |
|------|-------------|
| validation_error | Input validation failed |
| not_found | Resource not found |
| already_exists | Resource already exists |
| unauthorized | Authentication failed |
| forbidden | Insufficient permissions |
| rate_limit_exceeded | Too many requests |
```

---

### 5. Code Examples

```markdown
# Code Examples

## cURL
```bash
# List users
curl -H "Authorization: Bearer YOUR_API_KEY" \
  https://api.example.com/v1/users

# Create user
curl -X POST \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name":"John","email":"john@example.com"}' \
  https://api.example.com/v1/users
```

## JavaScript
```javascript
// List users
const response = await fetch('https://api.example.com/v1/users', {
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY'
  }
});
const users = await response.json();

// Create user
const response = await fetch('https://api.example.com/v1/users', {
  method: 'POST',
  headers: {
    'Authorization': 'Bearer YOUR_API_KEY',
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    name: 'John Doe',
    email: 'john@example.com'
  })
});
const user = await response.json();
```

## Python
```python
import requests

# List users
response = requests.get(
    'https://api.example.com/v1/users',
    headers={'Authorization': 'Bearer YOUR_API_KEY'}
)
users = response.json()

# Create user
response = requests.post(
    'https://api.example.com/v1/users',
    headers={'Authorization': 'Bearer YOUR_API_KEY'},
    json={'name': 'John Doe', 'email': 'john@example.com'}
)
user = response.json()
```

## Go
```go
// List users
req, _ := http.NewRequest("GET", "https://api.example.com/v1/users", nil)
req.Header.Set("Authorization", "Bearer YOUR_API_KEY")

client := &http.Client{}
resp, _ := client.Do(req)

// Create user
userData := map[string]string{
    "name": "John Doe",
    "email": "john@example.com",
}
body, _ := json.Marshal(userData)

req, _ = http.NewRequest("POST", "https://api.example.com/v1/users", bytes.NewBuffer(body))
req.Header.Set("Authorization", "Bearer YOUR_API_KEY")
req.Header.Set("Content-Type", "application/json")

resp, _ = client.Do(req)
```
```

---

## Interactive Documentation

### Swagger UI

```html
<!DOCTYPE html>
<html>
<head>
    <title>API Documentation</title>
    <link rel="stylesheet" href="https://unpkg.com/swagger-ui-dist/swagger-ui.css">
</head>
<body>
    <div id="swagger-ui"></div>
    
    <script src="https://unpkg.com/swagger-ui-dist/swagger-ui-bundle.js"></script>
    <script>
        SwaggerUIBundle({
            url: '/swagger.yaml',
            dom_id: '#swagger-ui',
            deepLinking: true,
            presets: [
                SwaggerUIBundle.presets.apis,
                SwaggerUIBundle.SwaggerUIStandalonePreset
            ]
        })
    </script>
</body>
</html>
```

**Features:**
- Try API directly in browser
- Auto-complete
- Response visualization
- Authentication support

---

### Postman Collection

```json
{
  "info": {
    "name": "User API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Users",
      "item": [
        {
          "name": "List Users",
          "request": {
            "method": "GET",
            "header": [
              {
                "key": "Authorization",
                "value": "Bearer {{api_key}}"
              }
            ],
            "url": {
              "raw": "{{base_url}}/users?page=1&limit=10",
              "host": ["{{base_url}}"],
              "path": ["users"],
              "query": [
                {"key": "page", "value": "1"},
                {"key": "limit", "value": "10"}
              ]
            }
          }
        },
        {
          "name": "Create User",
          "request": {
            "method": "POST",
            "header": [
              {
                "key": "Authorization",
                "value": "Bearer {{api_key}}"
              },
              {
                "key": "Content-Type",
                "value": "application/json"
              }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"name\": \"John Doe\",\n  \"email\": \"john@example.com\"\n}"
            },
            "url": {
              "raw": "{{base_url}}/users",
              "host": ["{{base_url}}"],
              "path": ["users"]
            }
          }
        }
      ]
    }
  ],
  "variable": [
    {
      "key": "base_url",
      "value": "https://api.example.com/v1"
    },
    {
      "key": "api_key",
      "value": "YOUR_API_KEY"
    }
  ]
}
```

---

## Documentation Best Practices

### 1. Keep It Updated

```
❌ Documentation from 2 years ago
✅ Auto-generated from code
✅ Updated with each release
```

---

### 2. Include Examples

```markdown
❌ Bad:
POST /users
Creates a user.

✅ Good:
POST /users
Creates a user.

Example:
```bash
curl -X POST \
  -d '{"name":"John","email":"john@example.com"}' \
  https://api.example.com/v1/users
```

Response:
```json
{
  "id": 123,
  "name": "John",
  "email": "john@example.com"
}
```
```

---

### 3. Document All Error Cases

```markdown
✅ Good:
### Errors
- `400` - Invalid email format
- `409` - Email already exists
- `422` - Validation failed
  ```json
  {
    "error": "validation_error",
    "details": {
      "email": "required"
    }
  }
  ```
```

---

### 4. Use Versioning

```markdown
# API Documentation v1.2.0

## Changelog

### v1.2.0 (2025-01-16)
- Added `/users/{id}/posts` endpoint
- Deprecated `phone` field (use `mobile` instead)

### v1.1.0 (2025-01-01)
- Added pagination to `/users`
- Fixed bug in date formatting
```

---

### 5. Search Functionality

```javascript
// Algolia DocSearch
<script src="https://cdn.jsdelivr.net/npm/docsearch.js@2/dist/cdn/docsearch.min.js"></script>
<script>
docsearch({
  apiKey: 'YOUR_API_KEY',
  indexName: 'your_docs',
  inputSelector: '#search',
});
</script>
```

---

## Documentation Tools

### Static Site Generators

**1. Docusaurus**
```bash
npx create-docusaurus@latest my-docs classic
```

**2. GitBook**
```bash
gitbook init
gitbook serve
```

**3. MkDocs**
```bash
pip install mkdocs
mkdocs new my-docs
mkdocs serve
```

---

### API-Specific Tools

**1. Redoc**
```html
<redoc spec-url="/swagger.yaml"></redoc>
```

**2. Stoplight**
- Visual API designer
- Mock servers
- Try it out

**3. Readme.io**
- Hosted documentation
- Interactive examples
- Analytics

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. __________ is the industry standard for REST API documentation.
2. Good documentation should include __________ examples for each endpoint.
3. Documentation should be __________ with each release.
4. The HTTP status code __________ indicates validation failed.

### True/False

1. ⬜ Documentation can be auto-generated from code
2. ⬜ Examples are optional in API documentation
3. ⬜ Swagger UI allows testing APIs in the browser
4. ⬜ Error codes don't need documentation
5. ⬜ Documentation should include all HTTP methods

### Multiple Choice

1. What's the best format for API documentation?
   - A) PDF
   - B) OpenAPI/Swagger
   - C) Plain text
   - D) Email

2. What should be in "Getting Started"?
   - A) Complete endpoint list
   - B) Authentication + quick example
   - C) Error codes
   - D) Changelog

3. Which allows interactive API testing?
   - A) Markdown files
   - B) Swagger UI
   - C) README
   - D) Comments

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. OpenAPI (or Swagger)
2. code
3. updated
4. 422

**True/False:**
1. ✅ True - Tools like Swaggo generate from comments
2. ❌ False - Examples are essential for usability
3. ✅ True - Swagger UI has "Try it out" feature
4. ❌ False - Error documentation is crucial
5. ✅ True - Document all supported methods

**Multiple Choice:**
1. **B** - OpenAPI/Swagger is industry standard
2. **B** - Authentication and quick example to get started
3. **B** - Swagger UI allows live API testing

</details>

</details>