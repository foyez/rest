# 4.4 Rate Limiting

## Why Rate Limiting?

**Without rate limiting:**
```
Attacker: 1,000,000 requests/second
→ Server crashes
→ Database overload
→ $10,000 cloud bill
→ Legitimate users can't access
```

**With rate limiting:**
```
Limit: 100 requests/minute per IP
Attacker blocked after 100
Server protected
Legitimate users unaffected
```

---

## Rate Limiting Strategies

### 1. Fixed Window

**Concept:** X requests per time window

```
Window: 1 minute
Limit: 100 requests

00:00 - 00:59: 100 requests allowed
01:00 - 01:59: Reset, 100 requests allowed
```

**Problem: Burst at boundary**
```
00:59: 100 requests
01:00: 100 requests
→ 200 requests in 1 second!
```

**Implementation:**

```go
type FixedWindowRateLimiter struct {
    requests map[string]int
    window   time.Duration
    limit    int
    mu       sync.Mutex
}

func (rl *FixedWindowRateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    currentWindow := time.Now().Truncate(rl.window)
    windowKey := fmt.Sprintf("%s:%d", key, currentWindow.Unix())
    
    count := rl.requests[windowKey]
    if count >= rl.limit {
        return false
    }
    
    rl.requests[windowKey]++
    return true
}
```

---

### 2. Sliding Window

**Concept:** Rolling time window

```
Current time: 00:30
Window: 1 minute

Count requests from 23:30 to 00:30
```

**Better:** No burst issue

**Implementation:**

```go
type SlidingWindowRateLimiter struct {
    requests map[string][]time.Time
    window   time.Duration
    limit    int
    mu       sync.Mutex
}

func (rl *SlidingWindowRateLimiter) Allow(key string) bool {
    rl.mu.Lock()
    defer rl.mu.Unlock()
    
    now := time.Now()
    windowStart := now.Add(-rl.window)
    
    // Remove old requests
    requests := rl.requests[key]
    validRequests := []time.Time{}
    for _, req := range requests {
        if req.After(windowStart) {
            validRequests = append(validRequests, req)
        }
    }
    
    if len(validRequests) >= rl.limit {
        return false
    }
    
    validRequests = append(validRequests, now)
    rl.requests[key] = validRequests
    
    return true
}
```

---

### 3. Token Bucket

**Concept:** Bucket refills with tokens

```
Bucket capacity: 100 tokens
Refill rate: 10 tokens/second

Request consumes 1 token
When empty, requests blocked
Refills over time
```

**Allows bursts:** If bucket full, can use all tokens at once

**Implementation:**

```go
type TokenBucket struct {
    tokens     float64
    capacity   float64
    refillRate float64
    lastRefill time.Time
    mu         sync.Mutex
}

func (tb *TokenBucket) Allow() bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()
    
    // Refill tokens
    now := time.Now()
    elapsed := now.Sub(tb.lastRefill).Seconds()
    tb.tokens += elapsed * tb.refillRate
    
    if tb.tokens > tb.capacity {
        tb.tokens = tb.capacity
    }
    
    tb.lastRefill = now
    
    // Check if token available
    if tb.tokens < 1 {
        return false
    }
    
    tb.tokens--
    return true
}
```

---

### 4. Leaky Bucket

**Concept:** Queue processes at fixed rate

```
Requests enter queue
Processed at constant rate
If queue full, reject
```

**Smooths traffic:** No bursts

---

## Rate Limit Headers

**Standard headers:**

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 75
X-RateLimit-Reset: 1737027600
```

**When limit exceeded:**

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1737027660

{
  "error": "rate_limit_exceeded",
  "message": "Too many requests. Please try again in 60 seconds.",
  "retry_after": 60
}
```

---

## Implementation with Redis

### Redis Commands

```redis
# Increment counter
INCR rate_limit:192.168.1.1:2025-01-16:10

# Set expiry
EXPIRE rate_limit:192.168.1.1:2025-01-16:10 60

# Get current count
GET rate_limit:192.168.1.1:2025-01-16:10
```

---

### Go Implementation

```go
package main

import (
    "context"
    "fmt"
    "net/http"
    "time"
    
    "github.com/go-redis/redis/v8"
)

var ctx = context.Background()

type RateLimiter struct {
    client *redis.Client
    limit  int
    window time.Duration
}

func NewRateLimiter(redisClient *redis.Client, limit int, window time.Duration) *RateLimiter {
    return &RateLimiter{
        client: redisClient,
        limit:  limit,
        window: window,
    }
}

func (rl *RateLimiter) Allow(key string) (bool, int, time.Time, error) {
    now := time.Now()
    windowKey := fmt.Sprintf("rate_limit:%s:%d", key, now.Unix()/int64(rl.window.Seconds()))
    
    // Increment counter
    count, err := rl.client.Incr(ctx, windowKey).Result()
    if err != nil {
        return false, 0, time.Time{}, err
    }
    
    // Set expiry on first request
    if count == 1 {
        rl.client.Expire(ctx, windowKey, rl.window)
    }
    
    // Calculate reset time
    resetTime := now.Truncate(rl.window).Add(rl.window)
    
    // Check limit
    remaining := rl.limit - int(count)
    if remaining < 0 {
        remaining = 0
    }
    
    allowed := count <= int64(rl.limit)
    
    return allowed, remaining, resetTime, nil
}

// Middleware
func RateLimitMiddleware(rl *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            // Use IP as key
            key := getClientIP(r)
            
            allowed, remaining, resetTime, err := rl.Allow(key)
            if err != nil {
                http.Error(w, "Internal server error", 500)
                return
            }
            
            // Set headers
            w.Header().Set("X-RateLimit-Limit", fmt.Sprintf("%d", rl.limit))
            w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", remaining))
            w.Header().Set("X-RateLimit-Reset", fmt.Sprintf("%d", resetTime.Unix()))
            
            if !allowed {
                retryAfter := int(time.Until(resetTime).Seconds())
                w.Header().Set("Retry-After", fmt.Sprintf("%d", retryAfter))
                
                w.WriteHeader(429)
                json.NewEncoder(w).Encode(map[string]interface{}{
                    "error":       "rate_limit_exceeded",
                    "message":     "Too many requests. Please try again later.",
                    "retry_after": retryAfter,
                })
                return
            }
            
            next.ServeHTTP(w, r)
        })
    }
}

func getClientIP(r *http.Request) string {
    // Check X-Forwarded-For header
    if ip := r.Header.Get("X-Forwarded-For"); ip != "" {
        return ip
    }
    
    // Check X-Real-IP header
    if ip := r.Header.Get("X-Real-IP"); ip != "" {
        return ip
    }
    
    // Use RemoteAddr
    return r.RemoteAddr
}

func main() {
    // Redis client
    rdb := redis.NewClient(&redis.Options{
        Addr: "localhost:6379",
    })
    
    // Rate limiter: 100 requests per minute
    rateLimiter := NewRateLimiter(rdb, 100, time.Minute)
    
    // Apply middleware
    http.Handle("/api/", RateLimitMiddleware(rateLimiter)(apiHandler))
    
    http.ListenAndServe(":8080", nil)
}
```

---

## Rate Limiting Strategies

### By IP Address

```go
key := getClientIP(r)
```

**Pros:** Simple
**Cons:** 
- Shared IPs (NAT, VPN)
- Easy to bypass (change IP)

---

### By User ID

```go
user := getCurrentUser(r)
key := fmt.Sprintf("user:%d", user.ID)
```

**Pros:** Accurate per-user limits
**Cons:** Requires authentication

---

### By API Key

```go
apiKey := r.Header.Get("X-API-Key")
key := fmt.Sprintf("api_key:%s", apiKey)
```

**Pros:** Fine-grained control
**Cons:** Requires API keys

---

### Tiered Limits

```go
func getRateLimit(user *User) int {
    switch user.Plan {
    case "free":
        return 100   // 100 req/hour
    case "pro":
        return 1000  // 1000 req/hour
    case "enterprise":
        return 10000 // 10000 req/hour
    default:
        return 10    // Anonymous: 10 req/hour
    }
}
```

---

## Different Limits for Different Endpoints

```go
func GetRateLimit(endpoint string) (int, time.Duration) {
    limits := map[string]struct{
        Limit  int
        Window time.Duration
    }{
        "/api/login":    {5, time.Minute},      // 5/min (prevent brute force)
        "/api/search":   {20, time.Minute},     // 20/min (expensive)
        "/api/users":    {100, time.Minute},    // 100/min (normal)
        "/api/health":   {1000, time.Minute},   // 1000/min (health check)
    }
    
    if limit, ok := limits[endpoint]; ok {
        return limit.Limit, limit.Window
    }
    
    return 100, time.Minute // Default
}
```

---

## Distributed Rate Limiting

**Problem:** Multiple servers

```
Server 1: User makes 100 requests
Server 2: User makes 100 requests
→ 200 requests total (limit bypassed!)
```

**Solution:** Centralized store (Redis)

```
All servers check Redis
Shared counter
Accurate limits
```

---

## Rate Limiting Best Practices

### 1. Return Proper Headers

```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 25
X-RateLimit-Reset: 1737027600
Retry-After: 60
```

### 2. Clear Error Messages

```json
{
  "error": "rate_limit_exceeded",
  "message": "You have exceeded the rate limit of 100 requests per minute.",
  "retry_after": 45,
  "limit": 100,
  "remaining": 0,
  "reset_at": "2025-01-16T11:00:00Z"
}
```

### 3. Different Limits for Different Actions

- Login: 5/min (prevent brute force)
- Search: 20/min (expensive operation)
- Read: 100/min (normal)
- Health check: No limit

### 4. Grace Period for Burst

Allow small bursts, then throttle.

### 5. Whitelist Internal IPs

```go
func isWhitelisted(ip string) bool {
    whitelisted := []string{
        "10.0.0.0/8",      // Internal network
        "192.168.0.0/16",  // Internal network
        "127.0.0.1",       // Localhost
    }
    
    // Check if IP in whitelist
    return contains(whitelisted, ip)
}
```

---

## Exponential Backoff (Client-Side)

**Client should retry with exponential backoff:**

```javascript
async function apiCallWithRetry(url, maxRetries = 3) {
    let retries = 0
    
    while (retries < maxRetries) {
        const response = await fetch(url)
        
        if (response.status === 429) {
            const retryAfter = response.headers.get('Retry-After')
            const delay = retryAfter ? parseInt(retryAfter) * 1000 : Math.pow(2, retries) * 1000
            
            console.log(`Rate limited. Retrying after ${delay}ms`)
            await sleep(delay)
            retries++
            continue
        }
        
        return response.json()
    }
    
    throw new Error('Max retries exceeded')
}
```

---

## Monitoring Rate Limits

```go
func (rl *RateLimiter) Allow(key string) bool {
    allowed := // ... check limit
    
    if !allowed {
        // Increment metric
        rateLimitHits.Inc()
        
        // Log
        log.Warn("Rate limit exceeded",
            "key", key,
            "limit", rl.limit,
        )
    }
    
    return allowed
}
```

**Alert when:**
- Spike in rate limit hits
- Specific user/IP constantly hitting limit
- Potential DDoS attack

---

## Real-World Examples

### GitHub API
```
60 requests/hour (unauthenticated)
5,000 requests/hour (authenticated)
```

### Twitter API
```
300 requests/15 minutes (user context)
450 requests/15 minutes (app context)
```

### Stripe API
```
100 read requests/second
100 write requests/second
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The HTTP status code for rate limiting is __________.
2. The __________ header tells clients when they can retry.
3. __________ rate limiting allows small bursts.
4. Rate limit counters should be stored in __________ for distributed systems.

### True/False

1. ⬜ Fixed window rate limiting has no burst issues
2. ⬜ Rate limit should return 403 Forbidden
3. ⬜ Retry-After header is required for 429 responses
4. ⬜ All endpoints should have the same rate limit
5. ⬜ Redis is good for distributed rate limiting

### Multiple Choice

1. Which rate limiting strategy allows bursts?
   - A) Fixed window
   - B) Sliding window
   - C) Token bucket
   - D) Leaky bucket

2. What should you include in rate limit responses?
   - A) Current limit
   - B) Remaining requests
   - C) Reset time
   - D) All of the above

3. What's the best key for rate limiting authenticated users?
   - A) IP address
   - B) User ID
   - C) API key
   - D) Session ID

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. 429
2. Retry-After
3. Token bucket
4. Redis

**True/False:**
1. ❌ False - Fixed window can have bursts at boundaries
2. ❌ False - Should return 429 Too Many Requests
3. ✅ True - Retry-After helps clients know when to retry
4. ❌ False - Different endpoints need different limits
5. ✅ True - Redis provides centralized, fast storage

**Multiple Choice:**
1. **C** - Token bucket allows bursts when tokens available
2. **D** - All information helps clients handle limits
3. **B** - User ID is most accurate for authenticated requests

</details>

</details>