# 8.1 Building a Secure API in Go

## Project: E-Commerce API

We'll build a production-ready e-commerce API with:
- User authentication (JWT)
- Product management
- Order processing
- Payment integration (Stripe)
- Rate limiting
- Logging
- Testing

---

## Project Structure

```
ecommerce-api/
├── cmd/
│   └── api/
│       └── main.go
├── internal/
│   ├── auth/
│   │   ├── jwt.go
│   │   └── middleware.go
│   ├── database/
│   │   └── postgres.go
│   ├── handlers/
│   │   ├── auth.go
│   │   ├── products.go
│   │   └── orders.go
│   ├── models/
│   │   ├── user.go
│   │   ├── product.go
│   │   └── order.go
│   └── middleware/
│       ├── logging.go
│       └── ratelimit.go
├── migrations/
│   └── 001_initial.sql
├── go.mod
└── README.md
```

---

## 1. Database Schema

```sql
-- migrations/001_initial.sql

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    name VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    stock INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INT REFERENCES users(id),
    total DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INT REFERENCES orders(id),
    product_id INT REFERENCES products(id),
    quantity INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
```

---

## 2. Models

```go
// internal/models/user.go
package models

import "time"

type User struct {
    ID           int       `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"` // Never send in JSON
    Name         string    `json:"name"`
    CreatedAt    time.Time `json:"created_at"`
}

type RegisterRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required,min=8"`
    Name     string `json:"name" validate:"required,min=3"`
}

type LoginRequest struct {
    Email    string `json:"email" validate:"required,email"`
    Password string `json:"password" validate:"required"`
}
```

```go
// internal/models/product.go
package models

import "time"

type Product struct {
    ID          int       `json:"id"`
    Name        string    `json:"name"`
    Description string    `json:"description"`
    Price       float64   `json:"price"`
    Stock       int       `json:"stock"`
    CreatedAt   time.Time `json:"created_at"`
}

type CreateProductRequest struct {
    Name        string  `json:"name" validate:"required,min=3"`
    Description string  `json:"description"`
    Price       float64 `json:"price" validate:"required,gt=0"`
    Stock       int     `json:"stock" validate:"required,gte=0"`
}
```

```go
// internal/models/order.go
package models

import "time"

type Order struct {
    ID        int         `json:"id"`
    UserID    int         `json:"user_id"`
    Total     float64     `json:"total"`
    Status    string      `json:"status"`
    Items     []OrderItem `json:"items"`
    CreatedAt time.Time   `json:"created_at"`
}

type OrderItem struct {
    ID        int     `json:"id"`
    ProductID int     `json:"product_id"`
    Quantity  int     `json:"quantity"`
    Price     float64 `json:"price"`
}

type CreateOrderRequest struct {
    Items []OrderItemRequest `json:"items" validate:"required,min=1,dive"`
}

type OrderItemRequest struct {
    ProductID int `json:"product_id" validate:"required"`
    Quantity  int `json:"quantity" validate:"required,min=1"`
}
```

---

## 3. Database Connection

```go
// internal/database/postgres.go
package database

import (
    "database/sql"
    "fmt"
    "time"
    
    _ "github.com/lib/pq"
)

type Config struct {
    Host     string
    Port     int
    User     string
    Password string
    DBName   string
    SSLMode  string
}

func NewPostgresDB(cfg Config) (*sql.DB, error) {
    connStr := fmt.Sprintf(
        "host=%s port=%d user=%s password=%s dbname=%s sslmode=%s",
        cfg.Host, cfg.Port, cfg.User, cfg.Password, cfg.DBName, cfg.SSLMode,
    )
    
    db, err := sql.Open("postgres", connStr)
    if err != nil {
        return nil, err
    }
    
    // Connection pool settings
    db.SetMaxOpenConns(25)
    db.SetMaxIdleConns(5)
    db.SetConnMaxLifetime(5 * time.Minute)
    
    // Test connection
    if err := db.Ping(); err != nil {
        return nil, err
    }
    
    return db, nil
}
```

---

## 4. JWT Authentication

```go
// internal/auth/jwt.go
package auth

import (
    "errors"
    "time"
    
    "github.com/golang-jwt/jwt/v5"
)

var jwtSecret = []byte("your-secret-key") // Use env variable in production

type Claims struct {
    UserID int    `json:"user_id"`
    Email  string `json:"email"`
    jwt.RegisteredClaims
}

func GenerateToken(userID int, email string) (string, error) {
    claims := Claims{
        UserID: userID,
        Email:  email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
            IssuedAt:  jwt.NewNumericDate(time.Now()),
        },
    }
    
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString(jwtSecret)
}

func ValidateToken(tokenString string) (*Claims, error) {
    token, err := jwt.ParseWithClaims(tokenString, &Claims{}, func(token *jwt.Token) (interface{}, error) {
        if _, ok := token.Method.(*jwt.SigningMethodHMAC); !ok {
            return nil, errors.New("invalid signing method")
        }
        return jwtSecret, nil
    })
    
    if err != nil {
        return nil, err
    }
    
    if claims, ok := token.Claims.(*Claims); ok && token.Valid {
        return claims, nil
    }
    
    return nil, errors.New("invalid token")
}
```

```go
// internal/auth/middleware.go
package auth

import (
    "context"
    "net/http"
    "strings"
)

func JWTMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        authHeader := r.Header.Get("Authorization")
        
        if authHeader == "" {
            http.Error(w, "Missing authorization header", http.StatusUnauthorized)
            return
        }
        
        parts := strings.Split(authHeader, " ")
        if len(parts) != 2 || parts[0] != "Bearer" {
            http.Error(w, "Invalid authorization header", http.StatusUnauthorized)
            return
        }
        
        claims, err := ValidateToken(parts[1])
        if err != nil {
            http.Error(w, "Invalid token", http.StatusUnauthorized)
            return
        }
        
        // Add claims to context
        ctx := context.WithValue(r.Context(), "user_id", claims.UserID)
        ctx = context.WithValue(ctx, "email", claims.Email)
        
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func GetUserID(r *http.Request) int {
    return r.Context().Value("user_id").(int)
}
```

---

## 5. Auth Handlers

```go
// internal/handlers/auth.go
package handlers

import (
    "database/sql"
    "encoding/json"
    "net/http"
    
    "github.com/go-playground/validator/v10"
    "golang.org/x/crypto/bcrypt"
    
    "your-app/internal/auth"
    "your-app/internal/models"
)

type AuthHandler struct {
    db       *sql.DB
    validate *validator.Validate
}

func NewAuthHandler(db *sql.DB) *AuthHandler {
    return &AuthHandler{
        db:       db,
        validate: validator.New(),
    }
}

func (h *AuthHandler) Register(w http.ResponseWriter, r *http.Request) {
    var req models.RegisterRequest
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Validate
    if err := h.validate.Struct(req); err != nil {
        http.Error(w, err.Error(), http.StatusUnprocessableEntity)
        return
    }
    
    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(req.Password), bcrypt.DefaultCost)
    if err != nil {
        http.Error(w, "Failed to hash password", http.StatusInternalServerError)
        return
    }
    
    // Insert user
    var userID int
    err = h.db.QueryRow(`
        INSERT INTO users (email, password_hash, name)
        VALUES ($1, $2, $3)
        RETURNING id
    `, req.Email, string(hashedPassword), req.Name).Scan(&userID)
    
    if err != nil {
        if strings.Contains(err.Error(), "unique constraint") {
            http.Error(w, "Email already exists", http.StatusConflict)
            return
        }
        http.Error(w, "Failed to create user", http.StatusInternalServerError)
        return
    }
    
    // Generate token
    token, err := auth.GenerateToken(userID, req.Email)
    if err != nil {
        http.Error(w, "Failed to generate token", http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "user_id": userID,
        "token":   token,
    })
}

func (h *AuthHandler) Login(w http.ResponseWriter, r *http.Request) {
    var req models.LoginRequest
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Get user
    var user models.User
    err := h.db.QueryRow(`
        SELECT id, email, password_hash, name
        FROM users
        WHERE email = $1
    `, req.Email).Scan(&user.ID, &user.Email, &user.PasswordHash, &user.Name)
    
    if err == sql.ErrNoRows {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }
    
    // Verify password
    if err := bcrypt.CompareHashAndPassword([]byte(user.PasswordHash), []byte(req.Password)); err != nil {
        http.Error(w, "Invalid credentials", http.StatusUnauthorized)
        return
    }
    
    // Generate token
    token, err := auth.GenerateToken(user.ID, user.Email)
    if err != nil {
        http.Error(w, "Failed to generate token", http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "token": token,
        "user":  user,
    })
}
```

---

## 6. Product Handlers

```go
// internal/handlers/products.go
package handlers

import (
    "database/sql"
    "encoding/json"
    "net/http"
    "strconv"
    
    "github.com/gorilla/mux"
    
    "your-app/internal/models"
)

type ProductHandler struct {
    db *sql.DB
}

func NewProductHandler(db *sql.DB) *ProductHandler {
    return &ProductHandler{db: db}
}

func (h *ProductHandler) List(w http.ResponseWriter, r *http.Request) {
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page < 1 {
        page = 1
    }
    
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit < 1 || limit > 100 {
        limit = 10
    }
    
    offset := (page - 1) * limit
    
    rows, err := h.db.Query(`
        SELECT id, name, description, price, stock, created_at
        FROM products
        ORDER BY created_at DESC
        LIMIT $1 OFFSET $2
    `, limit, offset)
    
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }
    defer rows.Close()
    
    products := []models.Product{}
    for rows.Next() {
        var p models.Product
        rows.Scan(&p.ID, &p.Name, &p.Description, &p.Price, &p.Stock, &p.CreatedAt)
        products = append(products, p)
    }
    
    // Get total count
    var total int
    h.db.QueryRow("SELECT COUNT(*) FROM products").Scan(&total)
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "data": products,
        "pagination": map[string]interface{}{
            "page":        page,
            "limit":       limit,
            "total":       total,
            "total_pages": (total + limit - 1) / limit,
        },
    })
}

func (h *ProductHandler) Get(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    
    var product models.Product
    err := h.db.QueryRow(`
        SELECT id, name, description, price, stock, created_at
        FROM products
        WHERE id = $1
    `, id).Scan(&product.ID, &product.Name, &product.Description,
        &product.Price, &product.Stock, &product.CreatedAt)
    
    if err == sql.ErrNoRows {
        http.Error(w, "Product not found", http.StatusNotFound)
        return
    }
    
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }
    
    json.NewEncoder(w).Encode(product)
}

func (h *ProductHandler) Create(w http.ResponseWriter, r *http.Request) {
    var req models.CreateProductRequest
    
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    var productID int
    err := h.db.QueryRow(`
        INSERT INTO products (name, description, price, stock)
        VALUES ($1, $2, $3, $4)
        RETURNING id
    `, req.Name, req.Description, req.Price, req.Stock).Scan(&productID)
    
    if err != nil {
        http.Error(w, "Failed to create product", http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]int{"id": productID})
}
```

---

## 7. Order Handlers

```go
// internal/handlers/orders.go
package handlers

import (
    "database/sql"
    "encoding/json"
    "net/http"
    
    "your-app/internal/auth"
    "your-app/internal/models"
)

type OrderHandler struct {
    db *sql.DB
}

func NewOrderHandler(db *sql.DB) *OrderHandler {
    return &OrderHandler{db: db}
}

func (h *OrderHandler) Create(w http.ResponseWriter, r *http.Request) {
    userID := auth.GetUserID(r)
    
    var req models.CreateOrderRequest
    if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
        http.Error(w, "Invalid request body", http.StatusBadRequest)
        return
    }
    
    // Start transaction
    tx, err := h.db.Begin()
    if err != nil {
        http.Error(w, "Failed to start transaction", http.StatusInternalServerError)
        return
    }
    defer tx.Rollback()
    
    // Calculate total and verify stock
    var total float64
    for _, item := range req.Items {
        var price float64
        var stock int
        
        err := tx.QueryRow(`
            SELECT price, stock FROM products WHERE id = $1 FOR UPDATE
        `, item.ProductID).Scan(&price, &stock)
        
        if err == sql.ErrNoRows {
            http.Error(w, "Product not found", http.StatusNotFound)
            return
        }
        
        if stock < item.Quantity {
            http.Error(w, "Insufficient stock", http.StatusConflict)
            return
        }
        
        total += price * float64(item.Quantity)
    }
    
    // Create order
    var orderID int
    err = tx.QueryRow(`
        INSERT INTO orders (user_id, total, status)
        VALUES ($1, $2, 'pending')
        RETURNING id
    `, userID, total).Scan(&orderID)
    
    if err != nil {
        http.Error(w, "Failed to create order", http.StatusInternalServerError)
        return
    }
    
    // Create order items and update stock
    for _, item := range req.Items {
        var price float64
        tx.QueryRow("SELECT price FROM products WHERE id = $1", item.ProductID).Scan(&price)
        
        _, err = tx.Exec(`
            INSERT INTO order_items (order_id, product_id, quantity, price)
            VALUES ($1, $2, $3, $4)
        `, orderID, item.ProductID, item.Quantity, price)
        
        if err != nil {
            http.Error(w, "Failed to create order item", http.StatusInternalServerError)
            return
        }
        
        // Update stock
        _, err = tx.Exec(`
            UPDATE products SET stock = stock - $1 WHERE id = $2
        `, item.Quantity, item.ProductID)
        
        if err != nil {
            http.Error(w, "Failed to update stock", http.StatusInternalServerError)
            return
        }
    }
    
    // Commit transaction
    if err := tx.Commit(); err != nil {
        http.Error(w, "Failed to commit transaction", http.StatusInternalServerError)
        return
    }
    
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(map[string]interface{}{
        "order_id": orderID,
        "total":    total,
    })
}

func (h *OrderHandler) List(w http.ResponseWriter, r *http.Request) {
    userID := auth.GetUserID(r)
    
    rows, err := h.db.Query(`
        SELECT id, total, status, created_at
        FROM orders
        WHERE user_id = $1
        ORDER BY created_at DESC
    `, userID)
    
    if err != nil {
        http.Error(w, "Database error", http.StatusInternalServerError)
        return
    }
    defer rows.Close()
    
    orders := []models.Order{}
    for rows.Next() {
        var o models.Order
        rows.Scan(&o.ID, &o.Total, &o.Status, &o.CreatedAt)
        orders = append(orders, o)
    }
    
    json.NewEncoder(w).Encode(orders)
}
```

---

## 8. Middleware

```go
// internal/middleware/logging.go
package middleware

import (
    "log"
    "net/http"
    "time"
)

type responseWriter struct {
    http.ResponseWriter
    statusCode int
}

func (rw *responseWriter) WriteHeader(code int) {
    rw.statusCode = code
    rw.ResponseWriter.WriteHeader(code)
}

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        rw := &responseWriter{w, http.StatusOK}
        
        next.ServeHTTP(rw, r)
        
        log.Printf(
            "%s %s %d %s",
            r.Method,
            r.RequestURI,
            rw.statusCode,
            time.Since(start),
        )
    })
}
```

```go
// internal/middleware/ratelimit.go
package middleware

import (
    "net/http"
    "sync"
    "time"
)

type visitor struct {
    count      int
    lastSeen   time.Time
}

var (
    visitors = make(map[string]*visitor)
    mu       sync.Mutex
)

func RateLimitMiddleware(requestsPerMinute int) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := r.RemoteAddr
            
            mu.Lock()
            v, exists := visitors[ip]
            
            if !exists {
                visitors[ip] = &visitor{count: 1, lastSeen: time.Now()}
                mu.Unlock()
                next.ServeHTTP(w, r)
                return
            }
            
            // Reset if minute has passed
            if time.Since(v.lastSeen) > time.Minute {
                v.count = 1
                v.lastSeen = time.Now()
                mu.Unlock()
                next.ServeHTTP(w, r)
                return
            }
            
            // Check limit
            if v.count >= requestsPerMinute {
                mu.Unlock()
                http.Error(w, "Rate limit exceeded", http.StatusTooManyRequests)
                return
            }
            
            v.count++
            mu.Unlock()
            
            next.ServeHTTP(w, r)
        })
    }
}
```

---

## 9. Main Application

```go
// cmd/api/main.go
package main

import (
    "log"
    "net/http"
    "os"
    
    "github.com/gorilla/mux"
    "github.com/joho/godotenv"
    
    "your-app/internal/auth"
    "your-app/internal/database"
    "your-app/internal/handlers"
    "your-app/internal/middleware"
)

func main() {
    // Load environment variables
    godotenv.Load()
    
    // Database connection
    db, err := database.NewPostgresDB(database.Config{
        Host:     os.Getenv("DB_HOST"),
        Port:     5432,
        User:     os.Getenv("DB_USER"),
        Password: os.Getenv("DB_PASSWORD"),
        DBName:   os.Getenv("DB_NAME"),
        SSLMode:  "disable",
    })
    
    if err != nil {
        log.Fatal("Failed to connect to database:", err)
    }
    defer db.Close()
    
    // Initialize handlers
    authHandler := handlers.NewAuthHandler(db)
    productHandler := handlers.NewProductHandler(db)
    orderHandler := handlers.NewOrderHandler(db)
    
    // Router
    r := mux.NewRouter()
    
    // Middleware
    r.Use(middleware.LoggingMiddleware)
    r.Use(middleware.RateLimitMiddleware(100))
    
    // Public routes
    r.HandleFunc("/register", authHandler.Register).Methods("POST")
    r.HandleFunc("/login", authHandler.Login).Methods("POST")
    r.HandleFunc("/products", productHandler.List).Methods("GET")
    r.HandleFunc("/products/{id}", productHandler.Get).Methods("GET")
    
    // Protected routes
    protected := r.PathPrefix("/").Subrouter()
    protected.Use(auth.JWTMiddleware)
    
    protected.HandleFunc("/products", productHandler.Create).Methods("POST")
    protected.HandleFunc("/orders", orderHandler.Create).Methods("POST")
    protected.HandleFunc("/orders", orderHandler.List).Methods("GET")
    
    // Start server
    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", r))
}
```

---

## 10. Environment Variables

```env
# .env
DB_HOST=localhost
DB_USER=postgres
DB_PASSWORD=password
DB_NAME=ecommerce

JWT_SECRET=your-secret-key-change-in-production

STRIPE_KEY=sk_test_...
```

---

## 11. Testing

```go
// internal/handlers/auth_test.go
package handlers

import (
    "bytes"
    "encoding/json"
    "net/http"
    "net/http/httptest"
    "testing"
)

func TestRegister(t *testing.T) {
    db := setupTestDB()
    defer db.Close()
    
    handler := NewAuthHandler(db)
    
    reqBody := map[string]string{
        "email":    "test@example.com",
        "password": "password123",
        "name":     "Test User",
    }
    
    body, _ := json.Marshal(reqBody)
    req := httptest.NewRequest("POST", "/register", bytes.NewBuffer(body))
    req.Header.Set("Content-Type", "application/json")
    
    rr := httptest.NewRecorder()
    
    handler.Register(rr, req)
    
    if rr.Code != http.StatusCreated {
        t.Errorf("Expected 201, got %d", rr.Code)
    }
    
    var response map[string]interface{}
    json.NewDecoder(rr.Body).Decode(&response)
    
    if response["token"] == nil {
        t.Error("Expected token in response")
    }
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. JWT tokens should be stored in the __________ header.
2. Passwords should never be stored in plain text, always __________.
3. Database transactions ensure __________ of operations.
4. Rate limiting prevents __________ attacks.

### True/False

1. ⬜ Middleware runs before route handlers
2. ⬜ JWT claims can contain sensitive data
3. ⬜ Transactions should be used for multi-step operations
4. ⬜ Password hashes should be returned in API responses
5. ⬜ Connection pooling improves database performance

### Multiple Choice

1. Where should JWT secret be stored?
   - A) In code
   - B) Environment variable
   - C) Database
   - D) Client-side

2. What's the purpose of bcrypt?
   - A) Encryption
   - B) Password hashing
   - C) Token generation
   - D) Data compression

3. When to use database transactions?
   - A) All queries
   - B) Multi-step operations
   - C) Read operations
   - D) Never

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. Authorization
2. hashed
3. atomicity (or consistency)
4. brute force (or DDoS)

**True/False:**
1. ✅ True - Middleware executes before handlers
2. ❌ False - JWT is base64 encoded, not encrypted
3. ✅ True - Transactions ensure all-or-nothing
4. ❌ False - Never expose password hashes
5. ✅ True - Reduces connection overhead

**Multiple Choice:**
1. **B** - Environment variable (never hardcode secrets)
2. **B** - Password hashing (slow + salted)
3. **B** - Multi-step operations (e.g., order + inventory)

</details>

</details>