# 6.4 GraphQL vs REST

## What is GraphQL?

**GraphQL** = Query language for APIs (developed by Facebook)

**Key difference:**
```
REST: Multiple endpoints, fixed responses
GraphQL: Single endpoint, flexible queries
```

---

## REST vs GraphQL

### REST Example

```
GET /users/123
Response:
{
  "id": 123,
  "name": "John",
  "email": "john@example.com",
  "address": { ... },
  "preferences": { ... }
}

GET /users/123/posts
Response:
[
  { "id": 1, "title": "Post 1" },
  { "id": 2, "title": "Post 2" }
]

GET /posts/1/comments
Response:
[
  { "id": 1, "text": "Great!" }
]

Total: 3 requests to get user + posts + comments
```

---

### GraphQL Example

```graphql
# Single request
{
  user(id: 123) {
    name
    email
    posts {
      title
      comments {
        text
      }
    }
  }
}

Response:
{
  "data": {
    "user": {
      "name": "John",
      "email": "john@example.com",
      "posts": [
        {
          "title": "Post 1",
          "comments": [
            { "text": "Great!" }
          ]
        }
      ]
    }
  }
}

Total: 1 request ✅
```

---

## Core Concepts

### 1. Schema

**REST:** No schema (or OpenAPI/Swagger)

**GraphQL:** Strongly typed schema required

```graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
  comments: [Comment!]!
}

type Comment {
  id: ID!
  text: String!
  author: User!
}
```

---

### 2. Queries

**Read data:**

```graphql
query {
  user(id: 123) {
    name
    email
  }
}
```

**With arguments:**

```graphql
query {
  posts(limit: 10, offset: 0) {
    id
    title
  }
}
```

**Nested queries:**

```graphql
query {
  user(id: 123) {
    name
    posts {
      title
      comments {
        text
        author {
          name
        }
      }
    }
  }
}
```

---

### 3. Mutations

**Create/Update/Delete:**

```graphql
mutation {
  createPost(title: "New Post", content: "Content") {
    id
    title
    createdAt
  }
}
```

```graphql
mutation {
  updateUser(id: 123, name: "New Name") {
    id
    name
  }
}
```

```graphql
mutation {
  deletePost(id: 456) {
    success
  }
}
```

---

### 4. Subscriptions

**Real-time updates:**

```graphql
subscription {
  newComment(postId: 123) {
    id
    text
    author {
      name
    }
  }
}
```

---

## Implementation (Go)

### GraphQL Server (gqlgen)

**1. Define Schema:**

```graphql
# schema.graphql
type Query {
  user(id: ID!): User
  users: [User!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
}

type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  author: User!
}

input CreateUserInput {
  name: String!
  email: String!
}
```

---

**2. Generate Code:**

```bash
go get github.com/99designs/gqlgen
gqlgen init
```

---

**3. Implement Resolvers:**

```go
package graph

import (
    "context"
    "github.com/yourapp/graph/model"
)

type Resolver struct {
    db *sql.DB
}

// Query resolvers
func (r *queryResolver) User(ctx context.Context, id string) (*model.User, error) {
    var user model.User
    
    err := r.db.QueryRow(`
        SELECT id, name, email
        FROM users
        WHERE id = ?
    `, id).Scan(&user.ID, &user.Name, &user.Email)
    
    if err != nil {
        return nil, err
    }
    
    return &user, nil
}

func (r *queryResolver) Users(ctx context.Context) ([]*model.User, error) {
    rows, err := r.db.Query("SELECT id, name, email FROM users")
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    users := []*model.User{}
    for rows.Next() {
        var user model.User
        rows.Scan(&user.ID, &user.Name, &user.Email)
        users = append(users, &user)
    }
    
    return users, nil
}

// Mutation resolvers
func (r *mutationResolver) CreateUser(ctx context.Context, input model.CreateUserInput) (*model.User, error) {
    result, err := r.db.Exec(`
        INSERT INTO users (name, email)
        VALUES (?, ?)
    `, input.Name, input.Email)
    
    if err != nil {
        return nil, err
    }
    
    id, _ := result.LastInsertId()
    
    return &model.User{
        ID:    fmt.Sprintf("%d", id),
        Name:  input.Name,
        Email: input.Email,
    }, nil
}

// Field resolvers (for nested data)
func (r *userResolver) Posts(ctx context.Context, obj *model.User) ([]*model.Post, error) {
    rows, err := r.db.Query(`
        SELECT id, title
        FROM posts
        WHERE user_id = ?
    `, obj.ID)
    
    if err != nil {
        return nil, err
    }
    defer rows.Close()
    
    posts := []*model.Post{}
    for rows.Next() {
        var post model.Post
        rows.Scan(&post.ID, &post.Title)
        posts = append(posts, &post)
    }
    
    return posts, nil
}
```

---

**4. Start Server:**

```go
package main

import (
    "log"
    "net/http"
    
    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/99designs/gqlgen/graphql/playground"
    "github.com/yourapp/graph"
)

func main() {
    srv := handler.NewDefaultServer(graph.NewExecutableSchema(graph.Config{
        Resolvers: &graph.Resolver{},
    }))
    
    http.Handle("/", playground.Handler("GraphQL playground", "/query"))
    http.Handle("/query", srv)
    
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## Advantages of GraphQL

### 1. No Over-fetching

**REST:**
```
GET /users/123
Returns: ALL user fields (even if you only need name)
```

**GraphQL:**
```graphql
{ user(id: 123) { name } }
Returns: Only name ✅
```

---

### 2. No Under-fetching

**REST:**
```
GET /users/123        → User data
GET /users/123/posts  → Posts
GET /posts/1/comments → Comments

3 requests needed
```

**GraphQL:**
```graphql
{
  user(id: 123) {
    name
    posts {
      title
      comments { text }
    }
  }
}

1 request ✅
```

---

### 3. Strongly Typed

```graphql
type User {
  id: ID!          # Non-null ID
  name: String!    # Non-null String
  age: Int         # Nullable Int
  posts: [Post!]!  # Non-null array of non-null Posts
}
```

---

### 4. Self-Documenting

GraphQL has introspection - automatic documentation!

```graphql
{
  __schema {
    types {
      name
      fields {
        name
        type {
          name
        }
      }
    }
  }
}
```

---

### 5. Versioning Not Needed

**REST:**
```
/v1/users
/v2/users  ← Breaking change
```

**GraphQL:**
```graphql
# Deprecate field
type User {
  oldField: String @deprecated(reason: "Use newField instead")
  newField: String
}
```

---

## Disadvantages of GraphQL

### 1. Complexity

**REST:** Simple, easy to cache

**GraphQL:** Complex queries, harder to cache

---

### 2. HTTP Caching Doesn't Work

**REST:**
```
GET /users/123
Cache-Control: max-age=3600
✅ Browsers/CDNs cache automatically
```

**GraphQL:**
```
POST /graphql
{ user(id: 123) { name } }
❌ Always POST, can't use HTTP cache
```

**Solution:** Use persisted queries or Apollo cache

---

### 3. N+1 Problem

**Problem:**

```graphql
{
  users {           # 1 query
    posts {         # N queries (one per user)
      comments {    # N*M queries
      }
    }
  }
}

100 users → 1 + 100 + (100 * posts) queries! 💥
```

**Solution:** DataLoader (batching)

```go
import "github.com/graph-gophers/dataloader"

func (r *Resolver) Posts(ctx context.Context, user *User) ([]*Post, error) {
    // Batch load posts for multiple users
    loader := ctx.Value("postLoader").(*dataloader.Loader)
    
    thunk := loader.Load(ctx, dataloader.StringKey(user.ID))
    result, err := thunk()
    
    return result.([]*Post), err
}
```

---

### 4. No File Upload (without extension)

**REST:**
```
POST /upload
Content-Type: multipart/form-data
✅ Native support
```

**GraphQL:**
```
Requires extension (graphql-upload)
```

---

### 5. Rate Limiting is Hard

**REST:**
```
Rate limit by endpoint:
/users → 100 req/min
/posts → 1000 req/min
```

**GraphQL:**
```
Single endpoint /graphql
How to rate limit complex queries?

Solution: Query complexity analysis
```

---

## When to Use REST

✅ **Use REST when:**

1. **Simple CRUD operations**
   ```
   GET /users
   POST /users
   PUT /users/123
   DELETE /users/123
   ```

2. **HTTP caching important**
   - Public data
   - CDN distribution
   - Browser caching

3. **File uploads/downloads**
   - Images
   - Videos
   - Documents

4. **Simple architecture**
   - Small team
   - Simple requirements
   - Quick to build

5. **Third-party integration**
   - Webhooks
   - OAuth callbacks
   - Payment gateways

---

## When to Use GraphQL

✅ **Use GraphQL when:**

1. **Mobile apps**
   - Minimize requests
   - Reduce data transfer
   - Battery optimization

2. **Complex data relationships**
   ```graphql
   {
     user {
       friends {
         posts {
           comments {
             likes
           }
         }
       }
     }
   }
   ```

3. **Multiple clients with different needs**
   - Web needs full data
   - Mobile needs minimal data
   - IoT needs specific fields

4. **Rapid iteration**
   - Frontend can change queries without backend changes
   - No versioning needed

5. **Real-time features**
   - Subscriptions
   - Live updates
   - Chat apps

---

## Hybrid Approach

**Many companies use both:**

```
REST:
- File uploads
- Webhooks
- OAuth
- Simple CRUD

GraphQL:
- Complex queries
- Mobile APIs
- Real-time subscriptions
```

**Example: GitHub**
- REST API v3: https://api.github.com/users/octocat
- GraphQL API v4: https://api.github.com/graphql

---

## Comparison Table

| Feature | REST | GraphQL |
|---------|------|---------|
| **Endpoints** | Multiple | Single |
| **Data Fetching** | Fixed | Flexible |
| **Over-fetching** | ❌ Common | ✅ No |
| **Under-fetching** | ❌ Common | ✅ No |
| **HTTP Caching** | ✅ Easy | ❌ Hard |
| **Versioning** | ❌ Needed | ✅ Not needed |
| **Learning Curve** | ✅ Easy | ❌ Steep |
| **File Upload** | ✅ Native | ⚠️ Extension |
| **Real-time** | ⚠️ SSE/WebSocket | ✅ Subscriptions |
| **Type Safety** | ⚠️ Optional | ✅ Built-in |
| **N+1 Problem** | ✅ No | ❌ Yes |
| **Rate Limiting** | ✅ Easy | ❌ Complex |

---

## Migration Strategy

### REST to GraphQL

**Don't rewrite everything!**

1. **Add GraphQL alongside REST**
   ```
   /api/rest/*     → REST endpoints
   /api/graphql    → GraphQL endpoint
   ```

2. **Wrap REST APIs**
   ```go
   func (r *queryResolver) User(ctx context.Context, id string) (*User, error) {
       // Call existing REST API
       resp, _ := http.Get(fmt.Sprintf("/api/users/%s", id))
       
       var user User
       json.NewDecoder(resp.Body).Decode(&user)
       
       return &user, nil
   }
   ```

3. **Migrate incrementally**
   ```
   Week 1: GraphQL for users
   Week 2: GraphQL for posts
   Week 3: GraphQL for comments
   ```

4. **Deprecate REST gradually**

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. GraphQL uses a __________ endpoint for all queries.
2. The __________ problem causes excessive database queries in GraphQL.
3. GraphQL queries are strongly __________.
4. REST is better for __________ caching than GraphQL.

### True/False

1. ⬜ GraphQL eliminates over-fetching
2. ⬜ GraphQL requires API versioning
3. ⬜ REST is better for file uploads
4. ⬜ GraphQL always performs better than REST
5. ⬜ GraphQL has built-in type system

### Multiple Choice

1. Main advantage of GraphQL?
   - A) Faster
   - B) Flexible queries
   - C) Easier to learn
   - D) Better caching

2. When to use REST over GraphQL?
   - A) Mobile apps
   - B) File uploads
   - C) Complex queries
   - D) Real-time updates

3. How to solve N+1 problem in GraphQL?
   - A) Use REST
   - B) DataLoader
   - C) More servers
   - D) Disable nesting

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. single
2. N+1
3. typed
4. HTTP

**True/False:**
1. ✅ True - Client requests only needed fields
2. ❌ False - Fields can be deprecated, no versioning needed
3. ✅ True - Native multipart/form-data support
4. ❌ False - Depends on use case; both have pros/cons
5. ✅ True - Schema defines types

**Multiple Choice:**
1. **B** - Flexible queries (request exactly what you need)
2. **B** - File uploads (native REST support)
3. **B** - DataLoader batches and caches queries

</details>

</details>