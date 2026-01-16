# 6.1 Idempotency

## What is Idempotency?

**Idempotent** = Operation that produces the same result no matter how many times it's executed

```
f(f(x)) = f(x)

Call once:  Result A
Call twice: Result A (same)
Call 100x:  Result A (same)
```

---

## Why Idempotency Matters

**Problem: Network issues**

```
Client sends: POST /orders
Network timeout...
Did it work? Unknown!

Client retries: POST /orders
Result: 2 orders created! 💰💰
```

**With idempotency:**
```
Client sends: POST /orders (idempotency-key: abc123)
Network timeout...

Client retries: POST /orders (idempotency-key: abc123)
Server: "Already processed, here's the original response"
Result: 1 order created ✅
```

---

## HTTP Methods and Idempotency

| Method | Idempotent | Safe | Example |
|--------|------------|------|---------|
| GET | ✅ Yes | ✅ Yes | Read data |
| PUT | ✅ Yes | ❌ No | Replace resource |
| DELETE | ✅ Yes | ❌ No | Delete resource |
| PATCH | ⚠️ Sometimes | ❌ No | Update fields |
| POST | ❌ No | ❌ No | Create resource |

---

## Naturally Idempotent Methods

### GET (Idempotent)

```
GET /users/123
GET /users/123
GET /users/123

Result: Always returns same user (if unchanged)
```

---

### PUT (Idempotent)

```
PUT /users/123
{
  "name": "John",
  "age": 30
}

PUT /users/123
{
  "name": "John",
  "age": 30
}

Result: User 123 has name="John", age=30
Same result whether called once or 100 times
```

---

### DELETE (Idempotent)

```
DELETE /users/123
Response: 204 No Content

DELETE /users/123
Response: 404 Not Found (already deleted)

Result: User 123 is gone
Whether you call it once or many times, user is deleted
```

---

## Non-Idempotent: POST

**Problem:**

```
POST /orders
{
  "product_id": 1,
  "quantity": 2
}

POST /orders (retry due to timeout)
{
  "product_id": 1,
  "quantity": 2
}

Result: 2 orders created! 😱
```

---

## Making POST Idempotent

### Solution: Idempotency Keys

**Concept:** Client generates unique key per operation

```
POST /orders
Idempotency-Key: a8f5k2p9m1q7z5x4

POST /orders (retry)
Idempotency-Key: a8f5k2p9m1q7z5x4  ← Same key

Server: "I already processed this key"
Returns original response (cached)
```

---

## Implementation

### Generate Idempotency Key (Client)

```javascript
// Frontend
function createOrder(orderData) {
    const idempotencyKey = crypto.randomUUID()
    
    return fetch('/api/orders', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json',
            'Idempotency-Key': idempotencyKey
        },
        body: JSON.stringify(orderData)
    })
}
```

---

### Server-Side Implementation (Go)

```go
package main

import (
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "net/http"
    "time"
)

type IdempotencyStore struct {
    cache map[string]IdempotencyRecord
}

type IdempotencyRecord struct {
    Response   []byte
    StatusCode int
    CreatedAt  time.Time
}

func (s *IdempotencyStore) Get(key string) (*IdempotencyRecord, bool) {
    record, exists := s.cache[key]
    
    // Expire after 24 hours
    if exists && time.Since(record.CreatedAt) > 24*time.Hour {
        delete(s.cache, key)
        return nil, false
    }
    
    return &record, exists
}

func (s *IdempotencyStore) Set(key string, response []byte, statusCode int) {
    s.cache[key] = IdempotencyRecord{
        Response:   response,
        StatusCode: statusCode,
        CreatedAt:  time.Now(),
    }
}

func IdempotencyMiddleware(store *IdempotencyStore) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Only for POST, PATCH (non-idempotent methods)
            if r.Method != "POST" && r.Method != "PATCH" {
                next.ServeHTTP(w, r)
                return
            }
            
            // Get idempotency key
            idempotencyKey := r.Header.Get("Idempotency-Key")
            if idempotencyKey == "" {
                http.Error(w, "Idempotency-Key header required", 400)
                return
            }
            
            // Check if already processed
            if record, exists := store.Get(idempotencyKey); exists {
                // Return cached response
                w.WriteHeader(record.StatusCode)
                w.Write(record.Response)
                return
            }
            
            // Capture response
            recorder := &ResponseRecorder{
                ResponseWriter: w,
                Body:           []byte{},
            }
            
            // Process request
            next.ServeHTTP(recorder, r)
            
            // Store response for future requests
            store.Set(idempotencyKey, recorder.Body, recorder.StatusCode)
        })
    }
}

type ResponseRecorder struct {
    http.ResponseWriter
    Body       []byte
    StatusCode int
}

func (r *ResponseRecorder) Write(b []byte) (int, error) {
    r.Body = append(r.Body, b...)
    return r.ResponseWriter.Write(b)
}

func (r *ResponseRecorder) WriteHeader(statusCode int) {
    r.StatusCode = statusCode
    r.ResponseWriter.WriteHeader(statusCode)
}
```

---

### Using Redis for Distributed Systems

```go
package main

import (
    "context"
    "time"
    
    "github.com/go-redis/redis/v8"
)

var ctx = context.Background()

type RedisIdempotencyStore struct {
    client *redis.Client
}

func (s *RedisIdempotencyStore) Get(key string) (string, bool) {
    val, err := s.client.Get(ctx, "idempotency:"+key).Result()
    if err == redis.Nil {
        return "", false
    }
    return val, true
}

func (s *RedisIdempotencyStore) Set(key, response string) {
    s.client.Set(ctx, "idempotency:"+key, response, 24*time.Hour)
}

func IdempotencyMiddleware(store *RedisIdempotencyStore) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            idempotencyKey := r.Header.Get("Idempotency-Key")
            
            if idempotencyKey == "" {
                http.Error(w, "Idempotency-Key required", 400)
                return
            }
            
            // Check cache
            if cachedResponse, exists := store.Get(idempotencyKey); exists {
                w.Header().Set("Content-Type", "application/json")
                w.Write([]byte(cachedResponse))
                return
            }
            
            // Process and cache
            recorder := &ResponseRecorder{ResponseWriter: w}
            next.ServeHTTP(recorder, r)
            
            store.Set(idempotencyKey, string(recorder.Body))
        })
    }
}
```

---

## Real-World Example: Payment Processing

```go
type Order struct {
    ID              int     `json:"id"`
    Amount          float64 `json:"amount"`
    IdempotencyKey  string  `json:"idempotency_key"`
}

func CreateOrder(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    idempotencyKey := r.Header.Get("Idempotency-Key")
    
    if idempotencyKey == "" {
        http.Error(w, "Idempotency-Key required", 400)
        return
    }
    
    // Check if order with this key exists
    var existingOrder Order
    err := db.QueryRow(`
        SELECT id, amount, idempotency_key
        FROM orders
        WHERE idempotency_key = ?
    `, idempotencyKey).Scan(&existingOrder.ID, &existingOrder.Amount, &existingOrder.IdempotencyKey)
    
    if err == nil {
        // Order already exists, return it
        w.WriteHeader(200)
        json.NewEncoder(w).Encode(existingOrder)
        return
    }
    
    // Parse request
    var req struct {
        Amount float64 `json:"amount"`
    }
    json.NewDecoder(r.Body).Decode(&req)
    
    // Create order
    result, err := db.Exec(`
        INSERT INTO orders (amount, idempotency_key)
        VALUES (?, ?)
    `, req.Amount, idempotencyKey)
    
    if err != nil {
        http.Error(w, "Failed to create order", 500)
        return
    }
    
    orderID, _ := result.LastInsertId()
    
    order := Order{
        ID:             int(orderID),
        Amount:         req.Amount,
        IdempotencyKey: idempotencyKey,
    }
    
    w.WriteHeader(201)
    json.NewEncoder(w).Encode(order)
}
```

---

## Idempotency Key Best Practices

### 1. Client-Generated

```javascript
✅ Client generates UUID
const key = crypto.randomUUID()

❌ Server generates
// Server can't guarantee same key on retry
```

---

### 2. Unique Per Operation

```javascript
❌ Same key for different operations
createOrder(data, key: "abc123")
createOrder(differentData, key: "abc123")

✅ Different key per operation
createOrder(data, key: crypto.randomUUID())
createOrder(differentData, key: crypto.randomUUID())
```

---

### 3. Include in Database

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    amount DECIMAL(10, 2),
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE UNIQUE INDEX idx_idempotency_key ON orders(idempotency_key);
```

---

### 4. Set Expiration

```go
// Store for 24 hours
redis.Set(ctx, key, response, 24*time.Hour)
```

---

### 5. Return Same Response

```go
// Don't recalculate
if cached, exists := cache.Get(key); exists {
    return cached  // Exact same response
}
```

---

## PATCH Idempotency

**Problem:** PATCH can be non-idempotent

```json
PATCH /users/123
{
  "balance": { "increment": 10 }
}

Call twice → Balance increases by 20 ❌
```

**Solution 1: Use absolute values**

```json
PATCH /users/123
{
  "balance": 110
}

Call multiple times → Balance = 110 ✅
```

**Solution 2: Use idempotency keys**

```
PATCH /users/123
Idempotency-Key: abc123
{
  "balance": { "increment": 10 }
}
```

---

## Conditional Requests (ETag)

**Alternative idempotency approach:**

```
GET /users/123
Response:
{
  "name": "John",
  "age": 30
}
ETag: "abc123"

PUT /users/123
If-Match: "abc123"
{
  "name": "John",
  "age": 31
}

If ETag changed (someone else updated):
Response: 412 Precondition Failed
```

---

## Stripe's Idempotency Implementation

**Real-world example:**

```bash
curl https://api.stripe.com/v1/charges \
  -H "Idempotency-Key: abc123xyz" \
  -d amount=2000 \
  -d currency=usd

# Retry with same key
curl https://api.stripe.com/v1/charges \
  -H "Idempotency-Key: abc123xyz" \
  -d amount=2000 \
  -d currency=usd

# Returns same charge, doesn't create duplicate
```

**Key features:**
- Keys stored for 24 hours
- Different request body with same key → 400 error
- Same key + same body → Returns original response

---

## Handling Edge Cases

### Concurrent Requests

**Problem:**
```
Request 1: POST /orders (key: abc123)
Request 2: POST /orders (key: abc123) ← Same time

Both check cache: Not found
Both create order: 2 orders! ❌
```

**Solution: Database constraint**

```sql
CREATE UNIQUE INDEX idx_idempotency ON orders(idempotency_key);
```

```go
result, err := db.Exec(`
    INSERT INTO orders (amount, idempotency_key)
    VALUES (?, ?)
`, amount, key)

if err != nil {
    // Check if duplicate key error
    if isDuplicateKeyError(err) {
        // Return existing order
        return getOrderByKey(key)
    }
    return err
}
```

---

### Different Request Bodies

**Stripe's approach:**

```
POST /charges (key: abc123)
Body: { amount: 2000 }

POST /charges (key: abc123)
Body: { amount: 3000 }  ← Different amount

Response: 400 Bad Request
"Idempotency key used with different parameters"
```

**Implementation:**

```go
type IdempotencyRecord struct {
    Response     []byte
    RequestHash  string  // Hash of request body
    StatusCode   int
}

func computeRequestHash(body []byte) string {
    hash := sha256.Sum256(body)
    return hex.EncodeToString(hash[:])
}

if record, exists := store.Get(key); exists {
    currentHash := computeRequestHash(requestBody)
    
    if record.RequestHash != currentHash {
        http.Error(w, "Idempotency key used with different parameters", 400)
        return
    }
    
    // Same request, return cached response
    return record.Response
}
```

---

## Testing Idempotency

```go
func TestIdempotency(t *testing.T) {
    key := "test-key-123"
    
    // First request
    resp1 := createOrder(key, orderData)
    order1 := parseResponse(resp1)
    
    // Second request (same key)
    resp2 := createOrder(key, orderData)
    order2 := parseResponse(resp2)
    
    // Should return same order ID
    if order1.ID != order2.ID {
        t.Error("Idempotency failed: different IDs")
    }
    
    // Should only create one record in database
    count := countOrdersByKey(key)
    if count != 1 {
        t.Errorf("Expected 1 order, got %d", count)
    }
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. An idempotent operation produces the same result when called __________ times.
2. __________ is naturally idempotent, while POST is not.
3. Idempotency keys should be generated by the __________.
4. Stripe stores idempotency keys for __________ hours.

### True/False

1. ⬜ GET requests are idempotent
2. ⬜ POST requests are idempotent
3. ⬜ DELETE requests are idempotent
4. ⬜ Idempotency keys should be server-generated
5. ⬜ Cached responses should be stored forever

### Multiple Choice

1. Which HTTP method is NOT idempotent?
   - A) GET
   - B) PUT
   - C) POST
   - D) DELETE

2. Who should generate idempotency keys?
   - A) Server
   - B) Client
   - C) Database
   - D) Load balancer

3. How long should idempotency keys be cached?
   - A) Forever
   - B) 1 hour
   - C) 24 hours
   - D) 1 year

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. multiple (or many)
2. PUT (or DELETE, GET)
3. client
4. 24

**True/False:**
1. ✅ True - GET is safe and idempotent
2. ❌ False - POST creates new resources each time
3. ✅ True - DELETE has same result after first call
4. ❌ False - Client should generate to ensure uniqueness on retry
5. ❌ False - Should expire (typically 24 hours)

**Multiple Choice:**
1. **C** - POST is not idempotent
2. **B** - Client generates unique keys
3. **C** - 24 hours is standard (Stripe, AWS)

</details>

</details>