# 6.2 Caching Strategies

## Why Cache?

**Without caching:**
```
Every request → Database query
1000 requests → 1000 DB queries
Slow response
High DB load
Expensive
```

**With caching:**
```
First request → Database → Cache
Next 999 requests → Cache (fast!)
Reduced DB load
Lower costs
```

---

## Cache Headers

### Cache-Control

```
Cache-Control: max-age=3600
→ Cache for 1 hour

Cache-Control: no-cache
→ Revalidate before using

Cache-Control: no-store
→ Don't cache at all

Cache-Control: public
→ Can be cached by CDN

Cache-Control: private
→ Only browser cache
```

---

### ETag (Entity Tag)

```
Request 1:
GET /users/123
Response:
ETag: "abc123"
{ "name": "John" }

Request 2:
GET /users/123
If-None-Match: "abc123"

Response (if unchanged):
304 Not Modified
(No body, saves bandwidth)
```

---

### Last-Modified

```
GET /users/123
Response:
Last-Modified: Wed, 15 Jan 2025 10:00:00 GMT

Next request:
GET /users/123
If-Modified-Since: Wed, 15 Jan 2025 10:00:00 GMT

Response (if not modified):
304 Not Modified
```

---

## Caching Layers

```
Client → CDN → Reverse Proxy → App Cache → Database
  ↓       ↓          ↓              ↓
Browser  Cloudflare  Nginx         Redis
Cache    Cache       Cache         Cache
```

---

## Application-Level Caching

### In-Memory Cache (Simple)

```go
type Cache struct {
    data map[string]interface{}
    mu   sync.RWMutex
}

func (c *Cache) Get(key string) (interface{}, bool) {
    c.mu.RLock()
    defer c.mu.RUnlock()
    
    val, exists := c.data[key]
    return val, exists
}

func (c *Cache) Set(key string, value interface{}) {
    c.mu.Lock()
    defer c.mu.Unlock()
    
    c.data[key] = value
}

// Usage
cache := &Cache{data: make(map[string]interface{})}

func GetUser(id int) (*User, error) {
    // Check cache
    if cached, exists := cache.Get(fmt.Sprintf("user:%d", id)); exists {
        return cached.(*User), nil
    }
    
    // Not in cache, fetch from DB
    user, err := db.GetUser(id)
    if err != nil {
        return nil, err
    }
    
    // Store in cache
    cache.Set(fmt.Sprintf("user:%d", id), user)
    
    return user, nil
}
```

---

### Redis Cache

```go
import (
    "context"
    "encoding/json"
    "time"
    
    "github.com/go-redis/redis/v8"
)

var ctx = context.Background()

func GetUser(rdb *redis.Client, db *sql.DB, id int) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    
    // Try cache
    cached, err := rdb.Get(ctx, key).Result()
    if err == nil {
        var user User
        json.Unmarshal([]byte(cached), &user)
        return &user, nil
    }
    
    // Cache miss, fetch from DB
    user, err := fetchUserFromDB(db, id)
    if err != nil {
        return nil, err
    }
    
    // Store in cache (expire in 1 hour)
    userData, _ := json.Marshal(user)
    rdb.Set(ctx, key, userData, 1*time.Hour)
    
    return user, nil
}
```

---

## Cache Strategies

### 1. Cache-Aside (Lazy Loading)

```
┌────────┐
│  App   │
└───┬────┘
    │
    ├─→ Check cache
    │   Cache miss?
    │
    ├─→ Read from DB
    │
    └─→ Store in cache
```

```go
func GetProduct(id int) (*Product, error) {
    // 1. Check cache
    if cached, exists := cache.Get(id); exists {
        return cached, nil
    }
    
    // 2. Cache miss - read from DB
    product, err := db.GetProduct(id)
    if err != nil {
        return nil, err
    }
    
    // 3. Store in cache
    cache.Set(id, product)
    
    return product, nil
}
```

**Pros:**
- Only cache what's needed
- Cache failure doesn't break app

**Cons:**
- First request always slow (cache miss)
- Potential cache stampede

---

### 2. Write-Through

```
┌────────┐
│  App   │
└───┬────┘
    │
    ├─→ Write to cache
    │
    └─→ Write to DB
```

```go
func UpdateProduct(product *Product) error {
    // 1. Write to cache
    cache.Set(product.ID, product)
    
    // 2. Write to DB
    err := db.UpdateProduct(product)
    if err != nil {
        cache.Delete(product.ID)  // Rollback cache
        return err
    }
    
    return nil
}
```

**Pros:**
- Cache always consistent
- Read requests fast

**Cons:**
- Write latency (2 operations)
- Unused data cached

---

### 3. Write-Behind (Write-Back)

```
┌────────┐
│  App   │
└───┬────┘
    │
    └─→ Write to cache
         ↓
    (Background job writes to DB)
```

```go
func UpdateProduct(product *Product) error {
    // Write to cache immediately
    cache.Set(product.ID, product)
    
    // Queue for async DB write
    writeQueue.Push(product)
    
    return nil
}

// Background worker
func processPendingWrites() {
    for product := range writeQueue {
        db.UpdateProduct(product)
    }
}
```

**Pros:**
- Fast writes
- Batch DB writes

**Cons:**
- Data loss risk (if cache fails before DB write)
- Complex

---

### 4. Refresh-Ahead

```
Before cache expires:
→ Refresh cache proactively
```

```go
func GetProduct(id int) (*Product, error) {
    cached, exists := cache.Get(id)
    
    if exists {
        // Check if expiring soon
        if cache.GetTTL(id) < 5*time.Minute {
            // Refresh in background
            go refreshCache(id)
        }
        
        return cached, nil
    }
    
    // Cache miss - load from DB
    product, _ := db.GetProduct(id)
    cache.Set(id, product, 1*time.Hour)
    
    return product, nil
}

func refreshCache(id int) {
    product, _ := db.GetProduct(id)
    cache.Set(id, product, 1*time.Hour)
}
```

---

## Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things." – Phil Karlton

### 1. Time-Based (TTL)

```go
// Expire after 1 hour
cache.Set(key, value, 1*time.Hour)
```

---

### 2. Event-Based

```go
func UpdateUser(user *User) error {
    err := db.UpdateUser(user)
    if err != nil {
        return err
    }
    
    // Invalidate cache
    cache.Delete(fmt.Sprintf("user:%d", user.ID))
    
    return nil
}
```

---

### 3. Tag-Based

```go
// Tag related items
cache.SetWithTags("user:123", user, []string{"users", "user:123"})
cache.SetWithTags("posts:user:123", posts, []string{"posts", "user:123"})

// Invalidate all caches for user 123
cache.InvalidateTag("user:123")
```

---

## Cache Stampede Prevention

**Problem:**

```
1000 concurrent requests
Cache expires
All hit DB simultaneously
DB overload! 💥
```

**Solution 1: Mutex (Lock)**

```go
var locks sync.Map

func GetProduct(id int) (*Product, error) {
    // Check cache
    if cached, exists := cache.Get(id); exists {
        return cached, nil
    }
    
    // Get or create lock for this ID
    lock, _ := locks.LoadOrStore(id, &sync.Mutex{})
    mu := lock.(*sync.Mutex)
    
    mu.Lock()
    defer mu.Unlock()
    
    // Double-check cache (another request might have loaded it)
    if cached, exists := cache.Get(id); exists {
        return cached, nil
    }
    
    // Load from DB
    product, _ := db.GetProduct(id)
    cache.Set(id, product, 1*time.Hour)
    
    return product, nil
}
```

---

**Solution 2: Probabilistic Early Expiration**

```go
import "math/rand"

func GetProduct(id int) (*Product, error) {
    cached, ttl := cache.GetWithTTL(id)
    
    if cached != nil {
        // Probabilistically refresh before expiry
        if rand.Float64() * float64(ttl) < 1 {
            go refreshCache(id)
        }
        
        return cached, nil
    }
    
    // Cache miss
    product, _ := db.GetProduct(id)
    cache.Set(id, product, 1*time.Hour)
    
    return product, nil
}
```

---

## HTTP Caching Headers Implementation

```go
func GetUser(w http.ResponseWriter, r *http.Request) {
    userID := mux.Vars(r)["id"]
    
    user, _ := db.GetUser(userID)
    
    // Calculate ETag
    etag := calculateETag(user)
    
    // Check If-None-Match header
    if r.Header.Get("If-None-Match") == etag {
        w.WriteHeader(304)  // Not Modified
        return
    }
    
    // Set cache headers
    w.Header().Set("ETag", etag)
    w.Header().Set("Cache-Control", "max-age=3600, public")
    
    json.NewEncoder(w).Encode(user)
}

func calculateETag(data interface{}) string {
    jsonData, _ := json.Marshal(data)
    hash := sha256.Sum256(jsonData)
    return fmt.Sprintf(`"%x"`, hash)
}
```

---

## Caching Patterns by Use Case

### Static Content (Images, CSS, JS)

```
Cache-Control: public, max-age=31536000, immutable
```

Long cache (1 year) + version in filename:
```
style.v123.css
app.v456.js
```

---

### API Responses (Rarely Change)

```
Cache-Control: public, max-age=3600
ETag: "abc123"
```

Cache for 1 hour, validate with ETag.

---

### User-Specific Data

```
Cache-Control: private, max-age=300
```

Browser cache only, 5 minutes.

---

### Real-Time Data

```
Cache-Control: no-cache
```

Always revalidate before using.

---

### Sensitive Data

```
Cache-Control: no-store
```

Never cache.

---

## Complete Example

```go
package main

import (
    "context"
    "crypto/sha256"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
    
    "github.com/go-redis/redis/v8"
    "github.com/gorilla/mux"
)

var ctx = context.Background()
var rdb *redis.Client

type Product struct {
    ID    int     `json:"id"`
    Name  string  `json:"name"`
    Price float64 `json:"price"`
}

func GetProduct(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    // Try cache first
    cacheKey := fmt.Sprintf("product:%s", id)
    cached, err := rdb.Get(ctx, cacheKey).Result()
    
    var product Product
    
    if err == nil {
        // Cache hit
        json.Unmarshal([]byte(cached), &product)
    } else {
        // Cache miss - fetch from DB
        product = fetchFromDB(id)
        
        // Store in cache (1 hour)
        productJSON, _ := json.Marshal(product)
        rdb.Set(ctx, cacheKey, productJSON, 1*time.Hour)
    }
    
    // Calculate ETag
    etag := calculateETag(product)
    
    // Check If-None-Match
    if r.Header.Get("If-None-Match") == etag {
        w.WriteHeader(304)
        return
    }
    
    // Set cache headers
    w.Header().Set("ETag", etag)
    w.Header().Set("Cache-Control", "public, max-age=3600")
    
    json.NewEncoder(w).Encode(product)
}

func UpdateProduct(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    var product Product
    json.NewDecoder(r.Body).Decode(&product)
    
    // Update DB
    updateDB(id, product)
    
    // Invalidate cache
    cacheKey := fmt.Sprintf("product:%s", id)
    rdb.Del(ctx, cacheKey)
    
    w.WriteHeader(204)
}

func calculateETag(data interface{}) string {
    jsonData, _ := json.Marshal(data)
    hash := sha256.Sum256(jsonData)
    return fmt.Sprintf(`"%x"`, hash)
}

func main() {
    rdb = redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    
    r := mux.NewRouter()
    r.HandleFunc("/products/{id}", GetProduct).Methods("GET")
    r.HandleFunc("/products/{id}", UpdateProduct).Methods("PUT")
    
    http.ListenAndServe(":8080", r)
}
```

---

## Best Practices

### 1. Cache Hot Data

```
80/20 rule: 20% of data gets 80% of requests
→ Cache that 20%
```

---

### 2. Set Appropriate TTL

```
Static content: 1 year
API responses: 1 hour
User data: 5 minutes
Real-time: No cache
```

---

### 3. Use Cache Keys Wisely

```
✅ user:123:profile
✅ product:456:details
✅ search:laptop:page:1

❌ user_data
❌ cache_123
```

---

### 4. Monitor Cache Hit Rate

```
Hit rate = Cache hits / (Cache hits + Cache misses)

Good: > 80%
Bad: < 50%
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. ETag returns __________ status code if content unchanged.
2. __________ strategy writes to cache first, then DB asynchronously.
3. Cache-Control: __________ prevents any caching.
4. The __________ problem occurs when cache expires and many requests hit DB.

### True/False

1. ⬜ Cache-Control: public allows CDN caching
2. ⬜ ETag prevents the need to send response body
3. ⬜ Write-through writes to DB first, then cache
4. ⬜ Cache stampede is prevented using locks
5. ⬜ Sensitive data should use Cache-Control: private

### Multiple Choice

1. What does 304 Not Modified mean?
   - A) Cache miss
   - B) Content unchanged, use cache
   - C) Cache expired
   - D) Cache disabled

2. Best cache strategy for frequently read data?
   - A) Write-through
   - B) Write-behind
   - C) Cache-aside
   - D) No cache

3. How to prevent cache stampede?
   - A) Disable cache
   - B) Use mutex/locks
   - C) Increase TTL
   - D) Use more servers

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 304
2. Write-behind (or Write-back)
3. no-store
4. cache stampede

**True/False:**
1. ✅ True - public allows shared caches like CDN
2. ✅ True - 304 response has no body
3. ❌ False - Write-through writes to both simultaneously
4. ✅ True - Locks ensure only one request loads data
5. ❌ False - Sensitive data should use no-store

**Multiple Choice:**
1. **B** - Content unchanged, client uses cached version
2. **C** - Cache-aside (lazy loading) for read-heavy
3. **B** - Mutex/locks prevent concurrent DB hits

</details>

</details>