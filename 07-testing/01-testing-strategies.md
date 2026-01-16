# 7.1 API Testing Strategies

## Why Test APIs?

**Without testing:**
```
Deploy → Bug in production → Users angry 😡
Lost revenue
Bad reputation
Emergency fix at 3am
```

**With testing:**
```
Write code → Test → Find bugs → Fix → Deploy → Happy users 😊
```

---

## Types of API Testing

```
┌─────────────────────────────────────┐
│ Manual Testing (Postman)            │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│ Unit Tests (Individual functions)   │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│ Integration Tests (Full endpoints)  │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│ E2E Tests (Complete workflows)      │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│ Load Tests (Performance)            │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│ Security Tests (Vulnerabilities)    │
└─────────────────────────────────────┘
```

---

## Unit Testing

**Test individual functions in isolation**

```go
package main

import (
    "testing"
)

// Function to test
func ValidateEmail(email string) bool {
    if email == "" {
        return false
    }
    
    // Simple validation
    return strings.Contains(email, "@")
}

// Unit test
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name     string
        email    string
        expected bool
    }{
        {"Valid email", "user@example.com", true},
        {"Missing @", "userexample.com", false},
        {"Empty string", "", false},
        {"Just @", "@", true},
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            result := ValidateEmail(tt.email)
            
            if result != tt.expected {
                t.Errorf("ValidateEmail(%s) = %v, expected %v",
                    tt.email, result, tt.expected)
            }
        })
    }
}
```

**Run tests:**
```bash
go test -v
```

---

## Integration Testing

**Test complete HTTP endpoints**

```go
package main

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestCreateUser(t *testing.T) {
    // Setup
    db := setupTestDB()
    defer db.Close()
    
    handler := CreateUserHandler(db)
    
    // Prepare request
    user := map[string]string{
        "name":  "John Doe",
        "email": "john@example.com",
    }
    
    body, _ := json.Marshal(user)
    req := httptest.NewRequest("POST", "/users", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    
    // Record response
    rr := httptest.NewRecorder()
    
    // Execute
    handler.ServeHTTP(rr, req)
    
    // Assert
    if rr.Code != http.StatusCreated {
        t.Errorf("Expected status 201, got %d", rr.Code)
    }
    
    var response map[string]interface{}
    json.NewDecoder(rr.Body).Decode(&response)
    
    if response["name"] != "John Doe" {
        t.Errorf("Expected name 'John Doe', got %v", response["name"])
    }
}
```

---

## Table-Driven Tests

**Test multiple scenarios efficiently**

```go
func TestGetUser(t *testing.T) {
    tests := []struct {
        name           string
        userID         string
        expectedStatus int
        expectedError  string
    }{
        {
            name:           "Valid user",
            userID:         "123",
            expectedStatus: 200,
            expectedError:  "",
        },
        {
            name:           "User not found",
            userID:         "999",
            expectedStatus: 404,
            expectedError:  "User not found",
        },
        {
            name:           "Invalid ID",
            userID:         "abc",
            expectedStatus: 400,
            expectedError:  "Invalid user ID",
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest("GET", "/users/"+tt.userID, nil)
            rr := httptest.NewRecorder()
            
            handler.ServeHTTP(rr, req)
            
            if rr.Code != tt.expectedStatus {
                t.Errorf("Expected status %d, got %d",
                    tt.expectedStatus, rr.Code)
            }
        })
    }
}
```

---

## Test Database Setup

### Option 1: In-Memory SQLite

```go
func setupTestDB() *sql.DB {
    db, _ := sql.Open("sqlite3", ":memory:")
    
    // Create tables
    db.Exec(`
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            name TEXT,
            email TEXT UNIQUE
        )
    `)
    
    return db
}
```

---

### Option 2: Test Database

```go
func setupTestDB() *sql.DB {
    db, _ := sql.Open("postgres",
        "postgres://user:pass@localhost/test_db?sslmode=disable")
    
    // Clean database
    db.Exec("TRUNCATE TABLE users CASCADE")
    
    return db
}

func teardownTestDB(db *sql.DB) {
    db.Exec("TRUNCATE TABLE users CASCADE")
    db.Close()
}
```

---

### Option 3: Docker Container

```go
func setupTestDB(t *testing.T) *sql.DB {
    // Start postgres container
    pool, _ := dockertest.NewPool("")
    
    resource, _ := pool.Run("postgres", "13", []string{
        "POSTGRES_PASSWORD=secret",
        "POSTGRES_DB=test_db",
    })
    
    var db *sql.DB
    if err := pool.Retry(func() error {
        var err error
        db, err = sql.Open("postgres",
            fmt.Sprintf("postgres://postgres:secret@localhost:%s/test_db?sslmode=disable",
                resource.GetPort("5432/tcp")))
        return db.Ping()
    }); err != nil {
        t.Fatal(err)
    }
    
    // Cleanup
    t.Cleanup(func() {
        pool.Purge(resource)
    })
    
    return db
}
```

---

## Mocking

**Mock external dependencies**

```go
// Interface for user repository
type UserRepository interface {
    GetUser(id int) (*User, error)
    CreateUser(user *User) error
}

// Mock implementation
type MockUserRepository struct {
    users map[int]*User
}

func (m *MockUserRepository) GetUser(id int) (*User, error) {
    user, exists := m.users[id]
    if !exists {
        return nil, errors.New("user not found")
    }
    return user, nil
}

func (m *MockUserRepository) CreateUser(user *User) error {
    m.users[user.ID] = user
    return nil
}

// Test with mock
func TestGetUserHandler(t *testing.T) {
    // Setup mock
    mockRepo := &MockUserRepository{
        users: map[int]*User{
            123: {ID: 123, Name: "John"},
        },
    }
    
    handler := NewUserHandler(mockRepo)
    
    // Test
    req := httptest.NewRequest("GET", "/users/123", nil)
    rr := httptest.NewRecorder()
    
    handler.ServeHTTP(rr, req)
    
    if rr.Code != 200 {
        t.Errorf("Expected 200, got %d", rr.Code)
    }
}
```

---

## Testing Authentication

```go
func TestProtectedEndpoint(t *testing.T) {
    tests := []struct {
        name           string
        token          string
        expectedStatus int
    }{
        {
            name:           "Valid token",
            token:          "valid-jwt-token",
            expectedStatus: 200,
        },
        {
            name:           "No token",
            token:          "",
            expectedStatus: 401,
        },
        {
            name:           "Invalid token",
            token:          "invalid-token",
            expectedStatus: 401,
        },
        {
            name:           "Expired token",
            token:          "expired-token",
            expectedStatus: 401,
        },
    }
    
    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            req := httptest.NewRequest("GET", "/protected", nil)
            
            if tt.token != "" {
                req.Header.Set("Authorization", "Bearer "+tt.token)
            }
            
            rr := httptest.NewRecorder()
            handler.ServeHTTP(rr, req)
            
            if rr.Code != tt.expectedStatus {
                t.Errorf("Expected %d, got %d",
                    tt.expectedStatus, rr.Code)
            }
        })
    }
}
```

---

## E2E Testing

**Test complete user workflows**

```go
func TestUserRegistrationFlow(t *testing.T) {
    // 1. Register user
    registerReq := map[string]string{
        "email":    "test@example.com",
        "password": "password123",
    }
    
    body, _ := json.Marshal(registerReq)
    resp := httpPost(t, "/register", body)
    
    if resp.StatusCode != 201 {
        t.Fatal("Registration failed")
    }
    
    var registerResp map[string]interface{}
    json.NewDecoder(resp.Body).Decode(&registerResp)
    
    // 2. Login with credentials
    loginReq := map[string]string{
        "email":    "test@example.com",
        "password": "password123",
    }
    
    body, _ = json.Marshal(loginReq)
    resp = httpPost(t, "/login", body)
    
    if resp.StatusCode != 200 {
        t.Fatal("Login failed")
    }
    
    var loginResp map[string]string
    json.NewDecoder(resp.Body).Decode(&loginResp)
    token := loginResp["token"]
    
    // 3. Access protected resource
    req := httptest.NewRequest("GET", "/profile", nil)
    req.Header.Set("Authorization", "Bearer "+token)
    
    rr := httptest.NewRecorder()
    handler.ServeHTTP(rr, req)
    
    if rr.Code != 200 {
        t.Fatal("Failed to access protected resource")
    }
}
```

---

## Load Testing

### Using Apache Bench

```bash
# 1000 requests, 10 concurrent
ab -n 1000 -c 10 http://localhost:8080/api/users

# With POST data
ab -n 1000 -c 10 -p data.json -T application/json \
   http://localhost:8080/api/users
```

---

### Using k6

```javascript
// load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    vus: 10,        // 10 virtual users
    duration: '30s', // 30 seconds
};

export default function() {
    let response = http.get('http://localhost:8080/api/users');
    
    check(response, {
        'status is 200': (r) => r.status === 200,
        'response time < 200ms': (r) => r.timings.duration < 200,
    });
    
    sleep(1);
}
```

```bash
k6 run load-test.js
```

---

### Using Go's testing package

```go
func BenchmarkGetUser(b *testing.B) {
    db := setupTestDB()
    defer db.Close()
    
    handler := GetUserHandler(db)
    
    b.ResetTimer()
    
    for i := 0; i < b.N; i++ {
        req := httptest.NewRequest("GET", "/users/123", nil)
        rr := httptest.NewRecorder()
        
        handler.ServeHTTP(rr, req)
    }
}
```

```bash
go test -bench=. -benchmem
```

---

## Security Testing

### 1. SQL Injection

```go
func TestSQLInjection(t *testing.T) {
    maliciousInputs := []string{
        "' OR '1'='1",
        "admin' --",
        "1'; DROP TABLE users; --",
    }
    
    for _, input := range maliciousInputs {
        req := httptest.NewRequest("GET", "/users?name="+input, nil)
        rr := httptest.NewRecorder()
        
        handler.ServeHTTP(rr, req)
        
        // Should not return all users or cause error
        if rr.Code == 200 {
            var users []User
            json.NewDecoder(rr.Body).Decode(&users)
            
            if len(users) > 1 {
                t.Errorf("SQL injection successful with input: %s", input)
            }
        }
    }
}
```

---

### 2. XSS Testing

```go
func TestXSSPrevention(t *testing.T) {
    xssPayloads := []string{
        "<script>alert('XSS')</script>",
        "<img src=x onerror=alert('XSS')>",
        "javascript:alert('XSS')",
    }
    
    for _, payload := range xssPayloads {
        user := map[string]string{
            "name": payload,
        }
        
        body, _ := json.Marshal(user)
        req := httptest.NewRequest("POST", "/users", bytes.NewBuffer(body))
        rr := httptest.NewRecorder()
        
        handler.ServeHTTP(rr, req)
        
        // Response should escape HTML
        if strings.Contains(rr.Body.String(), "<script>") {
            t.Error("XSS payload not escaped")
        }
    }
}
```

---

### 3. Rate Limiting

```go
func TestRateLimit(t *testing.T) {
    handler := RateLimitedHandler()
    
    // Make 100 requests
    for i := 0; i < 100; i++ {
        req := httptest.NewRequest("GET", "/api/users", nil)
        req.RemoteAddr = "192.168.1.1:1234"
        rr := httptest.NewRecorder()
        
        handler.ServeHTTP(rr, req)
    }
    
    // 101st request should be rate limited
    req := httptest.NewRequest("GET", "/api/users", nil)
    req.RemoteAddr = "192.168.1.1:1234"
    rr := httptest.NewRecorder()
    
    handler.ServeHTTP(rr, req)
    
    if rr.Code != 429 {
        t.Errorf("Expected 429 Too Many Requests, got %d", rr.Code)
    }
}
```

---

## Test Coverage

```bash
# Run tests with coverage
go test -cover

# Generate coverage report
go test -coverprofile=coverage.out
go tool cover -html=coverage.out
```

**Good coverage:**
```
Critical paths:     100%
Business logic:     >80%
Error handling:     >80%
Overall:            >70%
```

---

## Test Helpers

```go
// Helper: Create test request
func newRequest(t *testing.T, method, url string, body interface{}) *http.Request {
    var buf bytes.Buffer
    
    if body != nil {
        json.NewEncoder(&buf).Encode(body)
    }
    
    req := httptest.NewRequest(method, url, &buf)
    req.Header.Set("Content-Type", "application/json")
    
    return req
}

// Helper: Assert JSON response
func assertJSON(t *testing.T, rr *httptest.ResponseRecorder, expected interface{}) {
    var actual interface{}
    json.NewDecoder(rr.Body).Decode(&actual)
    
    if !reflect.DeepEqual(actual, expected) {
        t.Errorf("Response mismatch.\nExpected: %+v\nGot: %+v",
            expected, actual)
    }
}

// Usage
func TestExample(t *testing.T) {
    req := newRequest(t, "POST", "/users", map[string]string{
        "name": "John",
    })
    
    rr := httptest.NewRecorder()
    handler.ServeHTTP(rr, req)
    
    assertJSON(t, rr, map[string]string{
        "id":   "1",
        "name": "John",
    })
}
```

---

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: secret
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v2
      
      - name: Set up Go
        uses: actions/setup-go@v2
        with:
          go-version: 1.21
      
      - name: Run tests
        run: go test -v -cover ./...
        env:
          DB_HOST: localhost
          DB_PORT: 5432
          DB_USER: postgres
          DB_PASSWORD: secret
          DB_NAME: test_db
```

---

## Best Practices

### 1. Test Pyramid

```
        ╱╲
       ╱E2E╲       Few (slow, expensive)
      ╱─────╲
     ╱ Integ╲      Some (medium)
    ╱────────╲
   ╱   Unit   ╲    Many (fast, cheap)
  ╱────────────╲
```

**70% Unit, 20% Integration, 10% E2E**

---

### 2. Arrange-Act-Assert (AAA)

```go
func TestCreateUser(t *testing.T) {
    // Arrange
    db := setupTestDB()
    handler := CreateUserHandler(db)
    user := map[string]string{"name": "John"}
    
    // Act
    req := newRequest(t, "POST", "/users", user)
    rr := httptest.NewRecorder()
    handler.ServeHTTP(rr, req)
    
    // Assert
    if rr.Code != 201 {
        t.Errorf("Expected 201, got %d", rr.Code)
    }
}
```

---

### 3. Test Names

```go
✅ Good:
func TestGetUser_WhenUserExists_ReturnsUser(t *testing.T)
func TestGetUser_WhenUserNotFound_Returns404(t *testing.T)

❌ Bad:
func TestGetUser1(t *testing.T)
func TestGetUser2(t *testing.T)
```

---

### 4. Independent Tests

```go
❌ Bad (tests depend on each other):
func TestCreateUser(t *testing.T) {
    // Creates user with ID 1
}

func TestGetUser(t *testing.T) {
    // Assumes user 1 exists
}

✅ Good (each test is independent):
func TestGetUser(t *testing.T) {
    db := setupTestDB()
    // Create test data
    db.Exec("INSERT INTO users ...")
    
    // Test
    ...
    
    // Cleanup
    db.Exec("DELETE FROM users ...")
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. The test pyramid suggests __________ unit tests than E2E tests.
2. __________ allows you to fake external dependencies in tests.
3. Test coverage above __________% is considered good for critical paths.
4. The AAA pattern stands for Arrange, Act, __________.

### True/False

1. ⬜ Integration tests test individual functions
2. ⬜ E2E tests cover complete user workflows
3. ⬜ Tests should depend on each other for efficiency
4. ⬜ Mocking helps isolate code being tested
5. ⬜ 100% code coverage guarantees no bugs

### Multiple Choice

1. What's the best test for checking SQL injection?
   - A) Unit test
   - B) Integration test
   - C) Load test
   - D) Manual test

2. Which has most tests in the test pyramid?
   - A) E2E tests
   - B) Integration tests
   - C) Unit tests
   - D) Manual tests

3. What status code indicates rate limiting?
   - A) 400
   - B) 401
   - C) 429
   - D) 500

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. more
2. Mocking
3. 80 (or 100)
4. Assert

**True/False:**
1. ❌ False - Integration tests test complete endpoints
2. ✅ True - E2E tests simulate real user flows
3. ❌ False - Tests should be independent
4. ✅ True - Mocking isolates the code under test
5. ❌ False - Coverage helps but doesn't guarantee quality

**Multiple Choice:**
1. **B** - Integration test (tests actual endpoint with malicious input)
2. **C** - Unit tests (70% of test pyramid)
3. **C** - 429 Too Many Requests

</details>

</details>