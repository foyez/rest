# 6.3 Webhooks

## What are Webhooks?

**Webhook** = Reverse API - server calls your endpoint when something happens

**Traditional API (Polling):**
```
Client: "Any new orders?"
Server: "No"

Client: "Any new orders?"
Server: "No"

Client: "Any new orders?"
Server: "Yes! Here's order #123"

Problem: Wastes requests, delays, expensive
```

**Webhooks (Event-Driven):**
```
Client: "Call https://myapp.com/webhook when new order"
Server: "OK, registered"

[New order arrives]

Server → POST https://myapp.com/webhook
        { "event": "order.created", "order_id": 123 }

Instant notification! ✅
```

---

## How Webhooks Work

```
┌──────────────┐                           ┌──────────────┐
│   Service    │                           │   Your App   │
│  (Stripe,    │                           │              │
│   GitHub)    │                           │              │
└──────┬───────┘                           └──────┬───────┘
       │                                          │
       │ 1. Register webhook                      │
       │ POST /webhooks                           │
       │ { "url": "https://myapp.com/webhook" }   │
       │ <────────────────────────────────────────┤
       │                                          │
       │ 2. Event occurs (payment success)        │
       │                                          │
       │ 3. POST https://myapp.com/webhook        │
       │    { "event": "payment.success" }        │
       │ ──────────────────────────────────────>  │
       │                                          │
       │                                    4. Process
       │                                          │
       │ 5. 200 OK                                │
       │ <──────────────────────────────────────  │
```

---

## Real-World Examples

### Stripe (Payments)

```
Event: payment_intent.succeeded
Webhook payload:
{
  "id": "evt_1234",
  "type": "payment_intent.succeeded",
  "data": {
    "object": {
      "id": "pi_5678",
      "amount": 2000,
      "currency": "usd"
    }
  }
}
```

---

### GitHub (Code Push)

```
Event: push
Webhook payload:
{
  "ref": "refs/heads/main",
  "commits": [
    {
      "id": "abc123",
      "message": "Fix bug",
      "author": "john@example.com"
    }
  ]
}
```

---

### Slack (Message)

```
Event: message.channels
Webhook payload:
{
  "type": "message",
  "channel": "C1234567890",
  "user": "U2345678901",
  "text": "Hello World"
}
```

---

## Implementing Webhook Provider

### 1. Webhook Registration

```go
package main

import (
    "database/sql"
    "encoding/json"
    "net/http"
    "time"
)

type Webhook struct {
    ID        int       `json:"id"`
    URL       string    `json:"url"`
    Events    []string  `json:"events"`
    Secret    string    `json:"secret"`
    Active    bool      `json:"active"`
    CreatedAt time.Time `json:"created_at"`
}

// Register webhook
func RegisterWebhook(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    var webhook Webhook
    json.NewDecoder(r.Body).Decode(&webhook)
    
    // Validate URL
    if !isValidURL(webhook.URL) {
        http.Error(w, "Invalid webhook URL", 400)
        return
    }
    
    // Generate secret for signing
    secret := generateSecret()
    
    // Store in database
    result, err := db.Exec(`
        INSERT INTO webhooks (url, events, secret, active)
        VALUES (?, ?, ?, ?)
    `, webhook.URL, webhook.Events, secret, true)
    
    if err != nil {
        http.Error(w, "Failed to register webhook", 500)
        return
    }
    
    id, _ := result.LastInsertId()
    webhook.ID = int(id)
    webhook.Secret = secret
    
    w.WriteHeader(201)
    json.NewEncoder(w).Encode(webhook)
}
```

---

### 2. Sending Webhooks

```go
import (
    "bytes"
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "net/http"
)

type WebhookEvent struct {
    ID        string                 `json:"id"`
    Event     string                 `json:"event"`
    Data      map[string]interface{} `json:"data"`
    Timestamp time.Time              `json:"timestamp"`
}

func SendWebhook(webhook Webhook, event WebhookEvent) error {
    // Prepare payload
    payload, _ := json.Marshal(event)
    
    // Generate signature
    signature := generateSignature(payload, webhook.Secret)
    
    // Send HTTP request
    req, _ := http.NewRequest("POST", webhook.URL, bytes.NewBuffer(payload))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Webhook-Signature", signature)
    req.Header.Set("X-Webhook-Event", event.Event)
    
    client := &http.Client{
        Timeout: 10 * time.Second,
    }
    
    resp, err := client.Do(req)
    if err != nil {
        return err
    }
    defer resp.Body.Close()
    
    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        return nil
    }
    
    return fmt.Errorf("webhook failed with status %d", resp.StatusCode)
}

func generateSignature(payload []byte, secret string) string {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    return hex.EncodeToString(mac.Sum(nil))
}
```

---

### 3. Triggering Webhooks

```go
func CreateOrder(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    var order Order
    json.NewDecoder(r.Body).Decode(&order)
    
    // Save order to database
    orderID, err := saveOrder(db, order)
    if err != nil {
        http.Error(w, "Failed to create order", 500)
        return
    }
    
    // Trigger webhooks
    go triggerWebhooks(db, "order.created", map[string]interface{}{
        "order_id": orderID,
        "amount":   order.Amount,
        "status":   "pending",
    })
    
    w.WriteHeader(201)
    json.NewEncoder(w).Encode(map[string]int{"id": orderID})
}

func triggerWebhooks(db *sql.DB, eventType string, data map[string]interface{}) {
    // Get all webhooks subscribed to this event
    rows, _ := db.Query(`
        SELECT id, url, secret
        FROM webhooks
        WHERE active = true
        AND ? = ANY(events)
    `, eventType)
    
    webhooks := []Webhook{}
    for rows.Next() {
        var webhook Webhook
        rows.Scan(&webhook.ID, &webhook.URL, &webhook.Secret)
        webhooks = append(webhooks, webhook)
    }
    
    // Send to each webhook
    event := WebhookEvent{
        ID:        generateID(),
        Event:     eventType,
        Data:      data,
        Timestamp: time.Now(),
    }
    
    for _, webhook := range webhooks {
        go sendWithRetry(webhook, event)
    }
}
```

---

## Webhook Retry Logic

**Why retry?**
- Network issues
- Your server temporarily down
- Rate limiting

```go
func sendWithRetry(webhook Webhook, event WebhookEvent) {
    maxRetries := 3
    backoff := time.Second
    
    for i := 0; i < maxRetries; i++ {
        err := SendWebhook(webhook, event)
        
        if err == nil {
            // Success!
            logWebhookSuccess(webhook.ID, event.ID)
            return
        }
        
        // Failed, wait before retry
        logWebhookFailure(webhook.ID, event.ID, err)
        
        if i < maxRetries-1 {
            time.Sleep(backoff)
            backoff *= 2  // Exponential backoff
        }
    }
    
    // All retries failed
    disableWebhook(webhook.ID)
}
```

---

## Implementing Webhook Consumer

### 1. Receive Webhook

```go
func WebhookHandler(w http.ResponseWriter, r *http.Request) {
    // Read body
    body, err := ioutil.ReadAll(r.Body)
    if err != nil {
        http.Error(w, "Failed to read body", 400)
        return
    }
    
    // Verify signature
    signature := r.Header.Get("X-Webhook-Signature")
    if !verifySignature(body, signature) {
        http.Error(w, "Invalid signature", 401)
        return
    }
    
    // Parse event
    var event WebhookEvent
    json.Unmarshal(body, &event)
    
    // Process event
    go processEvent(event)
    
    // Respond quickly (don't wait for processing)
    w.WriteHeader(200)
}

func verifySignature(payload []byte, receivedSignature string) bool {
    secret := os.Getenv("WEBHOOK_SECRET")
    
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expectedSignature := hex.EncodeToString(mac.Sum(nil))
    
    return hmac.Equal([]byte(expectedSignature), []byte(receivedSignature))
}
```

---

### 2. Process Events

```go
func processEvent(event WebhookEvent) {
    switch event.Event {
    case "payment.succeeded":
        handlePaymentSuccess(event.Data)
    case "payment.failed":
        handlePaymentFailure(event.Data)
    case "order.created":
        handleOrderCreated(event.Data)
    default:
        log.Printf("Unknown event type: %s", event.Event)
    }
}

func handlePaymentSuccess(data map[string]interface{}) {
    paymentID := data["payment_id"].(string)
    amount := data["amount"].(float64)
    
    // Update order status
    db.Exec(`
        UPDATE orders
        SET status = 'paid', payment_id = ?
        WHERE id = ?
    `, paymentID, data["order_id"])
    
    // Send confirmation email
    sendEmail(data["email"].(string), "Payment Received", amount)
}
```

---

## Webhook Security

### 1. Signature Verification ✅

**Provider (sends webhook):**
```go
signature := hmac.SHA256(payload, secret)
headers["X-Webhook-Signature"] = signature
```

**Consumer (receives webhook):**
```go
expectedSignature := hmac.SHA256(payload, secret)
if signature != expectedSignature {
    return 401 Unauthorized
}
```

---

### 2. HTTPS Only

```go
func isValidURL(url string) bool {
    if !strings.HasPrefix(url, "https://") {
        return false
    }
    return true
}
```

---

### 3. Timestamp Validation

```go
type WebhookEvent struct {
    Timestamp time.Time `json:"timestamp"`
    // ...
}

func verifyTimestamp(timestamp time.Time) bool {
    // Reject if older than 5 minutes
    if time.Since(timestamp) > 5*time.Minute {
        return false
    }
    return true
}
```

---

### 4. IP Whitelist

```go
var allowedIPs = []string{
    "192.0.2.1",
    "198.51.100.0/24",
}

func isAllowedIP(ip string) bool {
    for _, allowed := range allowedIPs {
        if ip == allowed {
            return true
        }
    }
    return false
}
```

---

### 5. Idempotency

```go
func processEvent(event WebhookEvent) {
    // Check if already processed
    var count int
    db.QueryRow("SELECT COUNT(*) FROM processed_events WHERE event_id = ?", 
        event.ID).Scan(&count)
    
    if count > 0 {
        // Already processed, skip
        return
    }
    
    // Process event
    handleEvent(event)
    
    // Mark as processed
    db.Exec("INSERT INTO processed_events (event_id) VALUES (?)", event.ID)
}
```

---

## Webhook Best Practices

### 1. Respond Quickly

```go
❌ Don't do this:
func WebhookHandler(w http.ResponseWriter, r *http.Request) {
    event := parseEvent(r)
    processEvent(event)  // Takes 30 seconds
    w.WriteHeader(200)   // Provider timeout!
}

✅ Do this:
func WebhookHandler(w http.ResponseWriter, r *http.Request) {
    event := parseEvent(r)
    go processEvent(event)  // Process async
    w.WriteHeader(200)      // Respond immediately
}
```

---

### 2. Log Everything

```go
func logWebhook(event WebhookEvent, status string) {
    db.Exec(`
        INSERT INTO webhook_logs (event_id, event_type, status, payload)
        VALUES (?, ?, ?, ?)
    `, event.ID, event.Event, status, event)
}
```

---

### 3. Provide Retry Dashboard

```sql
CREATE TABLE webhook_deliveries (
    id SERIAL PRIMARY KEY,
    webhook_id INT,
    event_id VARCHAR(255),
    status VARCHAR(50),
    response_code INT,
    attempts INT,
    created_at TIMESTAMP,
    last_attempt_at TIMESTAMP
);
```

---

### 4. Support Webhook Testing

```go
// Test webhook endpoint
func TestWebhook(w http.ResponseWriter, r *http.Request) {
    webhookID := r.URL.Query().Get("id")
    
    // Get webhook
    webhook := getWebhook(webhookID)
    
    // Send test event
    testEvent := WebhookEvent{
        ID:    "test_" + generateID(),
        Event: "test",
        Data:  map[string]interface{}{"test": true},
    }
    
    err := SendWebhook(webhook, testEvent)
    
    if err != nil {
        http.Error(w, "Test failed: "+err.Error(), 500)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]string{
        "status": "success",
    })
}
```

---

## Complete Example

### Provider Side (Sends Webhooks)

```go
package main

import (
    "bytes"
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "net/http"
    "time"
)

type Webhook struct {
    ID     int
    URL    string
    Events []string
    Secret string
}

type Event struct {
    ID        string                 `json:"id"`
    Type      string                 `json:"type"`
    Data      map[string]interface{} `json:"data"`
    Timestamp time.Time              `json:"timestamp"`
}

func NotifyWebhooks(eventType string, data map[string]interface{}) {
    webhooks := getSubscribedWebhooks(eventType)
    
    event := Event{
        ID:        generateID(),
        Type:      eventType,
        Data:      data,
        Timestamp: time.Now(),
    }
    
    for _, webhook := range webhooks {
        go sendWebhook(webhook, event)
    }
}

func sendWebhook(webhook Webhook, event Event) error {
    payload, _ := json.Marshal(event)
    
    // Sign payload
    mac := hmac.New(sha256.New, []byte(webhook.Secret))
    mac.Write(payload)
    signature := hex.EncodeToString(mac.Sum(nil))
    
    // Send request
    req, _ := http.NewRequest("POST", webhook.URL, bytes.NewBuffer(payload))
    req.Header.Set("Content-Type", "application/json")
    req.Header.Set("X-Signature", signature)
    req.Header.Set("X-Event-Type", event.Type)
    
    client := &http.Client{Timeout: 10 * time.Second}
    resp, err := client.Do(req)
    
    if err != nil {
        return retry(webhook, event)
    }
    
    if resp.StatusCode >= 200 && resp.StatusCode < 300 {
        logSuccess(webhook.ID, event.ID)
        return nil
    }
    
    return retry(webhook, event)
}

func retry(webhook Webhook, event Event) error {
    // Retry logic with exponential backoff
    for i := 0; i < 3; i++ {
        time.Sleep(time.Duration(i+1) * time.Second)
        if err := sendWebhook(webhook, event); err == nil {
            return nil
        }
    }
    return fmt.Errorf("all retries failed")
}
```

---

### Consumer Side (Receives Webhooks)

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "io/ioutil"
    "net/http"
)

const webhookSecret = "your-secret-key"

type Event struct {
    ID        string                 `json:"id"`
    Type      string                 `json:"type"`
    Data      map[string]interface{} `json:"data"`
    Timestamp time.Time              `json:"timestamp"`
}

func WebhookHandler(w http.ResponseWriter, r *http.Request) {
    // Read payload
    payload, _ := ioutil.ReadAll(r.Body)
    
    // Verify signature
    signature := r.Header.Get("X-Signature")
    if !verifySignature(payload, signature) {
        http.Error(w, "Invalid signature", 401)
        return
    }
    
    // Parse event
    var event Event
    json.Unmarshal(payload, &event)
    
    // Process async
    go processEvent(event)
    
    // Respond immediately
    w.WriteHeader(200)
    w.Write([]byte(`{"status":"received"}`))
}

func verifySignature(payload []byte, signature string) bool {
    mac := hmac.New(sha256.New, []byte(webhookSecret))
    mac.Write(payload)
    expected := hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}

func processEvent(event Event) {
    // Check idempotency
    if isProcessed(event.ID) {
        return
    }
    
    // Handle event
    switch event.Type {
    case "payment.succeeded":
        handlePayment(event.Data)
    case "order.created":
        handleOrder(event.Data)
    }
    
    // Mark processed
    markProcessed(event.ID)
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Webhooks are __________ APIs where the server calls the client.
2. Webhook payloads should be signed using __________.
3. Webhook handlers should respond with __________ status code quickly.
4. Failed webhooks should be retried with __________ backoff.

### True/False

1. ⬜ Webhooks are more efficient than polling
2. ⬜ Webhook URLs can use HTTP
3. ⬜ Webhooks should process events synchronously before responding
4. ⬜ Signature verification prevents replay attacks
5. ⬜ Webhooks should implement retry logic

### Multiple Choice

1. What's the main advantage of webhooks over polling?
   - A) Simpler to implement
   - B) Real-time + less requests
   - C) More secure
   - D) Works offline

2. How should webhook receivers respond?
   - A) After processing event
   - B) Immediately (200 OK)
   - C) After validating signature
   - D) Never respond

3. What prevents webhook replay attacks?
   - A) HTTPS
   - B) Timestamp validation
   - C) IP whitelist
   - D) Rate limiting

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. reverse (or callback)
2. HMAC (or signature)
3. 200
4. exponential

**True/False:**
1. ✅ True - No wasted polling requests
2. ❌ False - HTTPS required for security
3. ❌ False - Process async, respond immediately
4. ✅ True - Signature ensures payload authenticity
5. ✅ True - Network failures require retries

**Multiple Choice:**
1. **B** - Real-time notifications with fewer requests
2. **B** - Respond immediately, process async
3. **B** - Timestamp prevents old webhooks from replaying

</details>

</details>