# 8.2 Common API Patterns

## Real-World Patterns You'll Encounter

This guide covers battle-tested patterns used by major companies like Stripe, GitHub, Twilio, and AWS.

---

## 1. Soft Delete Pattern

**Problem:** Need to "delete" records but keep data for audit/recovery

**Bad approach:**
```sql
DELETE FROM users WHERE id = 123;
-- Data gone forever! 😱
```

**Good approach (Soft Delete):**
```sql
-- Add deleted_at column
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMP;

-- "Delete" by setting timestamp
UPDATE users SET deleted_at = NOW() WHERE id = 123;

-- Queries exclude deleted records
SELECT * FROM users WHERE deleted_at IS NULL;
```

**Implementation:**

```go
type User struct {
    ID        int        `json:"id"`
    Email     string     `json:"email"`
    DeletedAt *time.Time `json:"-"` // Soft delete timestamp
}

// Soft delete
func (h *UserHandler) Delete(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    _, err := h.db.Exec(`
        UPDATE users
        SET deleted_at = NOW()
        WHERE id = $1 AND deleted_at IS NULL
    `, id)
    
    if err != nil {
        http.Error(w, "Failed to delete user", 500)
        return
    }
    
    w.WriteHeader(204)
}

// List only active users
func (h *UserHandler) List(w http.ResponseWriter, r *http.Request) {
    rows, _ := h.db.Query(`
        SELECT id, email
        FROM users
        WHERE deleted_at IS NULL
    `)
    // ...
}

// Admin: List all including deleted
func (h *UserHandler) ListAll(w http.ResponseWriter, r *http.Request) {
    includeDeleted := r.URL.Query().Get("include_deleted") == "true"
    
    query := "SELECT id, email, deleted_at FROM users"
    if !includeDeleted {
        query += " WHERE deleted_at IS NULL"
    }
    
    rows, _ := h.db.Query(query)
    // ...
}

// Restore deleted user
func (h *UserHandler) Restore(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    _, err := h.db.Exec(`
        UPDATE users
        SET deleted_at = NULL
        WHERE id = $1
    `, id)
    
    w.WriteHeader(200)
}
```

**Used by:** GitHub (repos), Stripe (customers), Shopify (products)

---

## 2. Optimistic Locking Pattern

**Problem:** Prevent concurrent updates from overwriting each other

**Scenario:**
```
User A: GET /products/1 → {price: 100, version: 1}
User B: GET /products/1 → {price: 100, version: 1}

User A: PUT /products/1 {price: 120} → Success (version: 2)
User B: PUT /products/1 {price: 110} → Overwrites A's change! ❌
```

**Solution: Version field**

```sql
ALTER TABLE products ADD COLUMN version INT DEFAULT 1;
```

```go
type Product struct {
    ID      int     `json:"id"`
    Name    string  `json:"name"`
    Price   float64 `json:"price"`
    Version int     `json:"version"`
}

func (h *ProductHandler) Update(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    var req struct {
        Price   float64 `json:"price"`
        Version int     `json:"version"`
    }
    json.NewDecoder(r.Body).Decode(&req)
    
    // Update only if version matches
    result, err := h.db.Exec(`
        UPDATE products
        SET price = $1, version = version + 1
        WHERE id = $2 AND version = $3
    `, req.Price, id, req.Version)
    
    if err != nil {
        http.Error(w, "Update failed", 500)
        return
    }
    
    rows, _ := result.RowsAffected()
    if rows == 0 {
        http.Error(w, "Conflict: Product was modified by another user", 409)
        return
    }
    
    w.WriteHeader(200)
}
```

**Alternative: ETags**
```go
func (h *ProductHandler) Update(w http.ResponseWriter, r *http.Request) {
    ifMatch := r.Header.Get("If-Match")
    
    // Get current ETag
    var currentETag string
    h.db.QueryRow("SELECT etag FROM products WHERE id = $1", id).Scan(&currentETag)
    
    if ifMatch != currentETag {
        http.Error(w, "Precondition Failed", 412)
        return
    }
    
    // Update with new ETag
    newETag := generateETag(updatedProduct)
    h.db.Exec("UPDATE products SET ..., etag = $1", newETag)
}
```

**Used by:** AWS DynamoDB, Azure Cosmos DB, Google Cloud Firestore

---

## 3. Bulk Operations Pattern

**Bad: N API calls**
```bash
POST /users {"name": "User 1"}
POST /users {"name": "User 2"}
POST /users {"name": "User 3"}
...
POST /users {"name": "User 100"}

100 network requests! 😱
```

**Good: Single bulk call**
```bash
POST /users/bulk
{
  "users": [
    {"name": "User 1"},
    {"name": "User 2"},
    ...
  ]
}
```

**Implementation:**

```go
func (h *UserHandler) BulkCreate(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Users []struct {
            Name  string `json:"name"`
            Email string `json:"email"`
        } `json:"users"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    // Limit bulk size
    if len(req.Users) > 1000 {
        http.Error(w, "Maximum 1000 users per request", 400)
        return
    }
    
    // Start transaction
    tx, _ := h.db.Begin()
    defer tx.Rollback()
    
    results := []map[string]interface{}{}
    
    for i, user := range req.Users {
        var userID int
        err := tx.QueryRow(`
            INSERT INTO users (name, email)
            VALUES ($1, $2)
            RETURNING id
        `, user.Name, user.Email).Scan(&userID)
        
        if err != nil {
            results = append(results, map[string]interface{}{
                "index":  i,
                "status": "error",
                "error":  err.Error(),
            })
            continue
        }
        
        results = append(results, map[string]interface{}{
            "index":   i,
            "status":  "success",
            "user_id": userID,
        })
    }
    
    tx.Commit()
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "results": results,
    })
}
```

**Alternative: Batch with partial success**
```go
// Return both successes and failures
{
  "successes": [
    {"index": 0, "id": 123},
    {"index": 2, "id": 125}
  ],
  "failures": [
    {"index": 1, "error": "Email already exists"}
  ]
}
```

**Used by:** Stripe (batch charges), SendGrid (bulk email), Twilio (bulk SMS)

---

## 4. Request Deduplication Pattern

**Problem:** Network issues cause duplicate requests

**Example:**
```
POST /orders {amount: 1000}
Network timeout...
User retries: POST /orders {amount: 1000}

Result: 2 orders charged! 💸💸
```

**Solution: Idempotency keys**

```go
func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    idempotencyKey := r.Header.Get("Idempotency-Key")
    
    if idempotencyKey == "" {
        http.Error(w, "Idempotency-Key required", 400)
        return
    }
    
    // Check if already processed
    var existingOrderID int
    err := h.db.QueryRow(`
        SELECT order_id
        FROM idempotent_requests
        WHERE idempotency_key = $1
    `, idempotencyKey).Scan(&existingOrderID)
    
    if err == nil {
        // Already processed, return existing order
        order := h.getOrder(existingOrderID)
        json.NewEncoder(w).Encode(order)
        return
    }
    
    // Process new order
    orderID := h.createOrder(r)
    
    // Store idempotency key
    h.db.Exec(`
        INSERT INTO idempotent_requests (idempotency_key, order_id)
        VALUES ($1, $2)
    `, idempotencyKey, orderID)
    
    json.NewEncoder(w).Encode(map[string]int{"order_id": orderID})
}
```

**Used by:** Stripe, PayPal, Square

---

## 5. Search Pattern

**Basic search (SQL LIKE):**
```go
func (h *ProductHandler) Search(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")
    
    rows, _ := h.db.Query(`
        SELECT id, name, description, price
        FROM products
        WHERE name ILIKE $1 OR description ILIKE $1
    `, "%"+query+"%")
    
    // Return results...
}
```

**Advanced: Full-text search**
```sql
-- Add full-text search index
ALTER TABLE products ADD COLUMN search_vector tsvector;

CREATE INDEX idx_search_vector ON products USING GIN(search_vector);

-- Update search vector
UPDATE products
SET search_vector = to_tsvector('english', name || ' ' || description);

-- Trigger to auto-update
CREATE TRIGGER update_search_vector
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW EXECUTE FUNCTION
  tsvector_update_trigger(search_vector, 'pg_catalog.english', name, description);
```

```go
func (h *ProductHandler) Search(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")
    
    rows, _ := h.db.Query(`
        SELECT id, name, description, price,
               ts_rank(search_vector, to_tsquery($1)) AS rank
        FROM products
        WHERE search_vector @@ to_tsquery($1)
        ORDER BY rank DESC
        LIMIT 20
    `, query)
    
    // Return results...
}
```

**With Elasticsearch:**
```go
import "github.com/elastic/go-elasticsearch/v8"

func (h *ProductHandler) Search(w http.ResponseWriter, r *http.Request) {
    query := r.URL.Query().Get("q")
    
    esQuery := map[string]interface{}{
        "query": map[string]interface{}{
            "multi_match": map[string]interface{}{
                "query":  query,
                "fields": []string{"name", "description"},
            },
        },
    }
    
    res, _ := h.es.Search(
        h.es.Search.WithIndex("products"),
        h.es.Search.WithBody(esReader(esQuery)),
    )
    
    // Parse and return results...
}
```

**Used by:** Amazon (product search), Airbnb (listing search), GitHub (code search)

---

## 6. Webhook Retry Pattern

**Problem:** Webhook delivery fails

**Solution: Exponential backoff**

```go
type Webhook struct {
    URL        string
    Event      string
    Payload    map[string]interface{}
    MaxRetries int
}

func (w *Webhook) Send() error {
    backoff := time.Second
    
    for attempt := 0; attempt < w.MaxRetries; attempt++ {
        err := w.sendHTTP()
        
        if err == nil {
            log.Printf("Webhook delivered on attempt %d", attempt+1)
            return nil
        }
        
        log.Printf("Webhook failed (attempt %d/%d): %v",
            attempt+1, w.MaxRetries, err)
        
        if attempt < w.MaxRetries-1 {
            time.Sleep(backoff)
            backoff *= 2 // Exponential: 1s, 2s, 4s, 8s...
        }
    }
    
    return errors.New("all retries failed")
}

func (w *Webhook) sendHTTP() error {
    payload, _ := json.Marshal(w.Payload)
    
    req, _ := http.NewRequest("POST", w.URL, bytes.NewBuffer(payload))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Event", w.Event)
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        return nil
    }
    
    return fmt.Errorf("webhook returned status %d", resp.StatusCode)
}
```

**Advanced: Dead letter queue**
```go
func (w *Webhook) Send() error {
    err := w.sendWithRetry()
    
    if err != nil {
        // Move to dead letter queue for manual processing
        deadLetterQueue.Push(w)
        
        // Notify administrators
        notifyAdmins("Webhook failed after all retries", w)
    }
    
    return err
}
```

**Used by:** Stripe, GitHub, Shopify

---

## 7. Multi-tenancy Pattern

**Problem:** Separate data for different customers/organizations

**Approach 1: Separate databases**
```go
func (h *Handler) getDB(tenantID string) *sql.DB {
    // Each tenant has own database
    return h.dbPool[tenantID]
}
```

**Approach 2: Shared database with tenant_id**
```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    name VARCHAR(255),
    price DECIMAL(10, 2),
    
    -- Ensure queries use this index
    INDEX idx_tenant_id (tenant_id)
);
```

```go
func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    tenantID := getTenantID(r)
    
    rows, _ := h.db.Query(`
        SELECT id, name, price
        FROM products
        WHERE tenant_id = $1
    `, tenantID)
    
    // Return results...
}
```

**Row-level security (PostgreSQL):**
```sql
-- Enable RLS
ALTER TABLE products ENABLE ROW LEVEL SECURITY;

-- Policy: Users see only their tenant's data
CREATE POLICY tenant_isolation ON products
    USING (tenant_id = current_setting('app.tenant_id')::int);

-- Set tenant in session
SET app.tenant_id = 123;

-- Now all queries automatically filtered!
SELECT * FROM products; -- Only returns tenant_id = 123
```

**Used by:** Salesforce, Slack, Shopify

---

## 8. Feature Flags Pattern

**Problem:** Enable/disable features without deploying

```go
type FeatureFlag struct {
    Name      string
    Enabled   bool
    Rollout   int // Percentage (0-100)
}

func (f *FeatureFlag) IsEnabled(userID int) bool {
    if !f.Enabled {
        return false
    }
    
    if f.Rollout == 100 {
        return true
    }
    
    // Gradual rollout based on user ID
    return (userID % 100) < f.Rollout
}

func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    userID := getUserID(r)
    
    newSearchEnabled := h.featureFlags["new_search"].IsEnabled(userID)
    
    if newSearchEnabled {
        // Use new search implementation
        h.newSearch(w, r)
    } else {
        // Use old search implementation
        h.oldSearch(w, r)
    }
}
```

**Database storage:**
```sql
CREATE TABLE feature_flags (
    name VARCHAR(255) PRIMARY KEY,
    enabled BOOLEAN DEFAULT false,
    rollout INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Used by:** Facebook, Netflix, Uber

---

## 9. Circuit Breaker Pattern

**Problem:** Cascading failures when service is down

```go
type CircuitBreaker struct {
    maxFailures  int
    timeout      time.Duration
    failures     int
    lastFailTime time.Time
    state        string // closed, open, half-open
}

func (cb *CircuitBreaker) Call(fn func() error) error {
    if cb.state == "open" {
        if time.Since(cb.lastFailTime) > cb.timeout {
            cb.state = "half-open"
        } else {
            return errors.New("circuit breaker is open")
        }
    }
    
    err := fn()
    
    if err != nil {
        cb.failures++
        cb.lastFailTime = time.Now()
        
        if cb.failures >= cb.maxFailures {
            cb.state = "open"
        }
        
        return err
    }
    
    // Success - reset
    cb.failures = 0
    cb.state = "closed"
    
    return nil
}

// Usage
func (h *PaymentHandler) Charge(w http.ResponseWriter, r *http.Request) {
    err := h.circuitBreaker.Call(func() error {
        return h.stripe.Charge(amount)
    })
    
    if err != nil {
        // Use fallback or queue for later
        h.queuePayment(amount)
        http.Error(w, "Payment queued", 202)
        return
    }
    
    w.WriteHeader(200)
}
```

**Used by:** Netflix (Hystrix), AWS, Uber

---

## 10. API Composition Pattern

**Problem:** Need data from multiple services

```go
type UserProfile struct {
    User     User     `json:"user"`
    Orders   []Order  `json:"orders"`
    Reviews  []Review `json:"reviews"`
}

func (h *ProfileHandler) Get(w http.ResponseWriter, r *http.Request) {
    userID := mux.Vars(r)["id"]
    
    // Fetch from multiple services in parallel
    var wg sync.WaitGroup
    var user User
    var orders []Order
    var reviews []Review
    
    wg.Add(3)
    
    go func() {
        defer wg.Done()
        user = h.userService.GetUser(userID)
    }()
    
    go func() {
        defer wg.Done()
        orders = h.orderService.GetUserOrders(userID)
    }()
    
    go func() {
        defer wg.Done()
        reviews = h.reviewService.GetUserReviews(userID)
    }()
    
    wg.Wait()
    
    profile := UserProfile{
        User:    user,
        Orders:  orders,
        Reviews: reviews,
    }
    
    json.NewEncoder(w).Encode(profile)
}
```

**Used by:** Netflix, Amazon, Uber Eats

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Soft delete uses a __________ column instead of actually deleting records.
2. Optimistic locking uses a __________ field to prevent concurrent updates.
3. Circuit breaker prevents __________ failures.
4. Feature flags enable __________ rollout of new features.

### True/False

1. ⬜ Soft delete permanently removes data
2. ⬜ Bulk operations reduce network requests
3. ⬜ Idempotency keys prevent duplicate operations
4. ⬜ Circuit breaker always retries failed requests
5. ⬜ Multi-tenancy requires separate databases

### Multiple Choice

1. What does soft delete do?
   - A) Deletes slowly
   - B) Marks as deleted
   - C) Deletes from memory
   - D) Deletes half the data

2. When to use optimistic locking?
   - A) All updates
   - B) Concurrent updates
   - C) Read operations
   - D) Never

3. What's the purpose of circuit breaker?
   - A) Break circuits
   - B) Stop cascading failures
   - C) Load balancing
   - D) Data encryption

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. deleted_at (or timestamp)
2. version (or etag)
3. cascading
4. gradual (or percentage-based)

**True/False:**
1. ❌ False - Soft delete marks records as deleted
2. ✅ True - Single request instead of many
3. ✅ True - Same key returns same result
4. ❌ False - Circuit breaker stops retrying when open
5. ❌ False - Can use shared database with tenant_id

**Multiple Choice:**
1. **B** - Marks records as deleted (timestamp)
2. **B** - Concurrent updates (prevent conflicts)
3. **B** - Stop cascading failures (fail fast)

</details>

</details>