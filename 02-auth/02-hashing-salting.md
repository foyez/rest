# 2.2 Hashing & Salting

## Why Hash Passwords?

**Never store passwords in plain text!**

```
❌ Database:
users
| id | email              | password    |
|----|--------------------|-----------  |
| 1  | john@example.com   | mypass123   |  ← DANGER!
| 2  | jane@example.com   | secret456   |  ← If hacked, all exposed
```

```
✅ Database:
users
| id | email              | password_hash                              |
|----|--------------------|--------------------------------------------|
| 1  | john@example.com   | $2a$10$N9qo8uL... |  ← Safe!
| 2  | jane@example.com   | $2a$10$3euPcmQ... |  ← Even if hacked, unusable
```

---

## Hashing vs Encryption

| Feature | Hashing | Encryption |
|---------|---------|------------|
| Reversible? | ❌ No (one-way) | ✅ Yes (two-way) |
| Same input | Same output | Different output (with different keys) |
| Use case | Passwords | Sensitive data you need to decrypt |
| Example | bcrypt, Argon2 | AES, RSA |

**Password flow:**
```
User enters: "mypassword123"
      ↓ Hash
Stored: "$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy"
      ↓ Cannot reverse!
```

---

## What is Salting?

**Salt** = Random string added to password before hashing

**Without salt:**
```
User 1: "password123" → Hash → abc123xyz
User 2: "password123" → Hash → abc123xyz  ← Same hash! (vulnerable)
```

**With salt:**
```
User 1: "password123" + "randomSalt1" → Hash → abc123xyz
User 2: "password123" + "randomSalt2" → Hash → def456uvw  ← Different!
```

**Why?**
- Prevents rainbow table attacks
- Same password = different hashes
- Each user has unique salt

---

## How Salting Works

```
Registration:
1. User submits: "mypassword"
2. Generate random salt: "a8f3k2p9"
3. Combine: "mypassword" + "a8f3k2p9"
4. Hash: bcrypt("mypassword" + "a8f3k2p9")
5. Store: hash in database (salt is included in hash)

Login:
1. User submits: "mypassword"
2. Retrieve hash from database
3. Extract salt from hash
4. Hash input: bcrypt("mypassword" + extracted_salt)
5. Compare: stored_hash == calculated_hash
```

---

## Rainbow Table Attack (Why Salt Matters)

**Without salt:**

Attacker pre-computes hashes of common passwords:

```
Rainbow Table:
password123 → 5f4dcc3b5aa765d61d8327deb882cf99
qwerty      → d8578edf8458ce06fbc5bb76a58c5ca4
admin       → 21232f297a57a5a743894a0e4a801fc3
```

If your database leaks:
```
User hash: 5f4dcc3b5aa765d61d8327deb882cf99
Lookup in table: "password123" ← FOUND!
```

**With salt:**

```
User 1: password123 + salt1 → unique_hash_1
User 2: password123 + salt2 → unique_hash_2
User 3: password123 + salt3 → unique_hash_3
```

Rainbow table is useless! Each hash is unique.

---

## Hashing Algorithms

### ❌ Never Use

**MD5, SHA-1, SHA-256**
```go
❌ hash := md5.Sum([]byte(password))  // Too fast!
❌ hash := sha256.Sum256([]byte(password))  // Too fast!
```

**Why avoid?**
- Too fast (billions of hashes/second)
- GPUs can crack quickly
- No built-in salt

---

### ✅ Use These

#### 1. bcrypt (Most Common)

```go
import "golang.org/x/crypto/bcrypt"

// Hash password
func HashPassword(password string) (string, error) {
    // Cost = 10 (recommended, higher = slower = more secure)
    hash, err := bcrypt.GenerateFromPassword([]byte(password), 10)
    if err != nil {
        return "", err
    }
    return string(hash), nil
}

// Verify password
func CheckPassword(password, hash string) bool {
    err := bcrypt.CompareHashAndPassword([]byte(hash), []byte(password))
    return err == nil
}
```

**Usage:**
```go
// Registration
hashedPassword, _ := HashPassword("userPassword123")
SaveUser(email, hashedPassword)

// Login
user := GetUserByEmail(email)
if CheckPassword(inputPassword, user.PasswordHash) {
    // Login successful
} else {
    // Invalid password
}
```

**Pros:**
- Widely adopted
- Automatic salt generation
- Adjustable cost (work factor)
- Battle-tested

**Cons:**
- Password length limited to 72 bytes
- Slower than Argon2

---

#### 2. Argon2 (Most Secure)

```go
import "golang.org/x/crypto/argon2"

func HashPasswordArgon2(password string) (string, error) {
    salt := make([]byte, 16)
    _, err := rand.Read(salt)
    if err != nil {
        return "", err
    }
    
    // Time=1, Memory=64MB, Threads=4, KeyLen=32
    hash := argon2.IDKey([]byte(password), salt, 1, 64*1024, 4, 32)
    
    // Encode salt + hash
    encoded := base64.StdEncoding.EncodeToString(salt) + ":" +
                base64.StdEncoding.EncodeToString(hash)
    
    return encoded, nil
}

func VerifyPasswordArgon2(password, encoded string) bool {
    parts := strings.Split(encoded, ":")
    salt, _ := base64.StdEncoding.DecodeString(parts[0])
    storedHash, _ := base64.StdEncoding.DecodeString(parts[1])
    
    hash := argon2.IDKey([]byte(password), salt, 1, 64*1024, 4, 32)
    
    return bytes.Equal(hash, storedHash)
}
```

**Pros:**
- Most secure (winner of Password Hashing Competition 2015)
- Resistant to GPU/ASIC attacks
- Memory-hard (uses lots of RAM)

**Cons:**
- Newer (less battle-tested than bcrypt)
- More complex to implement

---

#### 3. scrypt

**Good but less common than bcrypt/Argon2.**

---

## Complete Example: User Registration & Login

```go
package main

import (
    "database/sql"
    "errors"
    "golang.org/x/crypto/bcrypt"
)

type User struct {
    ID           int
    Email        string
    PasswordHash string
}

// Register new user
func RegisterUser(db *sql.DB, email, password string) error {
    // Validate password strength
    if len(password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    
    // Hash password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), 10)
    if err != nil {
        return err
    }
    
    // Save to database
    _, err = db.Exec(
        "INSERT INTO users (email, password_hash) VALUES (?, ?)",
        email,
        string(hashedPassword),
    )
    
    return err
}

// Login user
func LoginUser(db *sql.DB, email, password string) (*User, error) {
    var user User
    
    // Get user by email
    err := db.QueryRow(
        "SELECT id, email, password_hash FROM users WHERE email = ?",
        email,
    ).Scan(&user.ID, &user.Email, &user.PasswordHash)
    
    if err != nil {
        return nil, errors.New("invalid email or password")
    }
    
    // Verify password
    err = bcrypt.CompareHashAndPassword(
        []byte(user.PasswordHash),
        []byte(password),
    )
    
    if err != nil {
        return nil, errors.New("invalid email or password")
    }
    
    return &user, nil
}
```

**API endpoints:**
```go
func RegisterHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    err := RegisterUser(db, req.Email, req.Password)
    if err != nil {
        http.Error(w, err.Error(), 400)
        return
    }
    
    json.NewEncoder(w).Encode(map[string]string{
        "message": "User created successfully",
    })
}

func LoginHandler(w http.ResponseWriter, r *http.Request) {
    var req struct {
        Email    string `json:"email"`
        Password string `json:"password"`
    }
    
    json.NewDecoder(r.Body).Decode(&req)
    
    user, err := LoginUser(db, req.Email, req.Password)
    if err != nil {
        http.Error(w, "Invalid email or password", 401)
        return
    }
    
    // Generate JWT token
    token := GenerateJWT(user.ID)
    
    json.NewEncoder(w).Encode(map[string]interface{}{
        "token": token,
        "user": user,
    })
}
```

---

## Password Strength Validation

```go
func ValidatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    
    var (
        hasUpper   = false
        hasLower   = false
        hasNumber  = false
        hasSpecial = false
    )
    
    for _, char := range password {
        switch {
        case unicode.IsUpper(char):
            hasUpper = true
        case unicode.IsLower(char):
            hasLower = true
        case unicode.IsNumber(char):
            hasNumber = true
        case unicode.IsPunct(char) || unicode.IsSymbol(char):
            hasSpecial = true
        }
    }
    
    if !hasUpper {
        return errors.New("password must contain uppercase letter")
    }
    if !hasLower {
        return errors.New("password must contain lowercase letter")
    }
    if !hasNumber {
        return errors.New("password must contain number")
    }
    if !hasSpecial {
        return errors.New("password must contain special character")
    }
    
    return nil
}
```

---

## Common Mistakes

### ❌ Using MD5/SHA-256

```go
❌ hash := md5.Sum([]byte(password))
```

**Why bad?**
- Too fast (attacker can try billions/second)
- No salt
- Designed for file integrity, not passwords

---

### ❌ Implementing your own hashing

```go
❌ func myHash(password string) string {
    return sha256.Sum256([]byte(password + "mysalt"))
}
```

**Why bad?**
- Crypto is hard
- Easy to introduce vulnerabilities
- Use battle-tested libraries

---

### ❌ Storing salt separately

```go
❌ users table:
| id | email | password_hash | salt |
```

**Why bad?**
- bcrypt includes salt in hash automatically
- Extra column unnecessary
- More attack surface

**✅ bcrypt hash contains everything:**
```
$2a$10$N9qo8uLOickgx2ZMRZoMye...
│  │  │   └─ Hash
│  │  └────── Salt
│  └─────────── Cost
└────────────── Algorithm
```

---

### ❌ Logging passwords

```go
❌ log.Info("User login attempt", "email", email, "password", password)
```

**Never log:**
- Passwords (even before hashing)
- Password hashes
- Tokens
- Session IDs

---

## Best Practices

1. ✅ **Use bcrypt or Argon2**
2. ✅ **Never store plain text passwords**
3. ✅ **Use strong cost factor** (bcrypt cost >= 10)
4. ✅ **Validate password strength** (length, complexity)
5. ✅ **Implement rate limiting** on login
6. ✅ **Use HTTPS** for login endpoints
7. ✅ **Generic error messages** ("Invalid email or password")
8. ✅ **Consider password managers** (allow long passwords)

---

## Timing Attack Protection

**❌ Vulnerable:**
```go
func CheckEmail(email string) bool {
    user := GetUserByEmail(email)
    if user == nil {
        return false  // Returns fast if email doesn't exist
    }
    // More processing...
}
```

Attacker can measure response time to find valid emails.

**✅ Protected:**
```go
func Login(email, password string) error {
    user := GetUserByEmail(email)
    
    // Always hash password, even if user doesn't exist
    if user == nil {
        bcrypt.CompareHashAndPassword(
            []byte("$2a$10$dummy..."),  // Dummy hash
            []byte(password),
        )
        return ErrInvalidCredentials
    }
    
    err := bcrypt.CompareHashAndPassword(
        []byte(user.PasswordHash),
        []byte(password),
    )
    
    if err != nil {
        return ErrInvalidCredentials
    }
    
    return nil
}
```

Same execution time whether user exists or not.

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. A __________ is a random string added to a password before hashing.
2. __________ is a one-way function, while encryption is two-way.
3. The recommended bcrypt cost factor is at least __________.
4. __________ and __________ are secure password hashing algorithms (name two).

### True/False

1. ⬜ MD5 is a secure password hashing algorithm
2. ⬜ Salts prevent rainbow table attacks
3. ⬜ bcrypt automatically generates and includes salt in the hash
4. ⬜ You should store the salt in a separate database column
5. ⬜ Hashing is reversible if you know the algorithm

### Multiple Choice

1. Why should passwords be hashed instead of encrypted?
   - A) Hashing is faster
   - B) Hashing is irreversible (one-way)
   - C) Encryption requires a key
   - D) Both B and C

2. What attack does salting prevent?
   - A) SQL injection
   - B) Rainbow table attack
   - C) XSS attack
   - D) CSRF attack

3. Which is the most secure password hashing algorithm?
   - A) MD5
   - B) SHA-256
   - C) bcrypt
   - D) Argon2

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. salt
2. Hashing
3. 10
4. bcrypt, Argon2 (or scrypt)

**True/False:**
1. ❌ False - MD5 is too fast and insecure for passwords
2. ✅ True - Each password gets unique salt, making rainbow tables useless
3. ✅ True - bcrypt hash includes algorithm, cost, salt, and hash
4. ❌ False - bcrypt includes salt in hash; separate column unnecessary
5. ❌ False - Hashing is one-way and irreversible

**Multiple Choice:**
1. **D** - Hashing is irreversible (can't recover password) and no key management needed
2. **B** - Salting makes each hash unique, defeating pre-computed rainbow tables
3. **D** - Argon2 is the most secure (won Password Hashing Competition 2015)

</details>

</details>

---

## Interview Questions

<details>
<summary><strong>View Questions</strong></summary>

### Question 1: Why should you never use MD5 or SHA-256 for password hashing?

### Question 2: Explain how salting works and why it's important

### Question 3: How does bcrypt work? Why is it better than simple hashing?

### Question 4: Implement user registration and login with bcrypt in Go

### Question 5: How would you handle password resets securely?

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

### Answer 1: Why not MD5/SHA-256

**Three main problems:**

**1. Too fast**
```
MD5/SHA-256: Billions of hashes per second on GPU
bcrypt: ~1000 hashes per second
```

**Speed comparison:**
- MD5: 3,800,000,000 hashes/second (GPU)
- SHA-256: 2,000,000,000 hashes/second (GPU)
- bcrypt: ~1,000 hashes/second

**Why speed is bad:**
- Attacker can brute force quickly
- 8-character password: cracked in hours

**2. No built-in salt**
```go
❌ sha256.Sum256([]byte(password))
// Same password = same hash
```

**3. Designed for different purpose**
- MD5/SHA: File integrity, checksums
- bcrypt/Argon2: Password hashing

**Example attack:**

```
Leaked database:
user1: 5f4dcc3b5aa765d61d8327deb882cf99
user2: 5f4dcc3b5aa765d61d8327deb882cf99

Attacker:
1. Run MD5 on common passwords
2. "password" → 5f4dcc3b5aa765d61d8327deb882cf99
3. Both users have password "password" ← CRACKED!
```

**With bcrypt:**
```
user1: $2a$10$N9qo8uLOickgx2ZMRZoMye...
user2: $2a$10$3euPcmQFCOcNJkTFTF.KLu...

Same password, different hashes!
Attacker must crack each individually
Takes years instead of hours
```

---

### Answer 2: How salting works

**Purpose:** Make same password hash differently for each user

**Process:**

**1. Generate random salt**
```go
salt := generateRandomBytes(16)
// Example: "a8f3k2p9m1q7z5x4"
```

**2. Combine password + salt**
```
User password: "mypassword"
Salt: "a8f3k2p9"
Combined: "mypassword" + "a8f3k2p9" = "mypassworda8f3k2p9"
```

**3. Hash the combination**
```go
hash := bcrypt(combined)
// Result: $2a$10$N9qo8uLOickgx2ZMRZoMye...
```

**4. Store hash (salt included)**
```
Database:
password_hash: $2a$10$N9qo8uLOickgx2ZMRZoMye...
                     ↑   ↑
                     │   └─ Hash
                     └───── Salt (base64 encoded)
```

**Verification:**

```
1. User submits: "mypassword"
2. Retrieve stored hash
3. Extract salt from hash
4. Hash input with same salt
5. Compare hashes
```

**Why it's important:**

**Without salt - Rainbow Table Attack:**
```
Attacker pre-computes:
password123 → abc123
admin       → def456
qwerty      → ghi789

If database leaks:
User hash: abc123
Lookup: "password123" ← FOUND in milliseconds!
```

**With salt:**
```
User 1: password123 + salt1 → hash1
User 2: password123 + salt2 → hash2  ← Different!
User 3: password123 + salt3 → hash3  ← Different!

Rainbow table is useless
Must crack each hash individually
```

**Real numbers:**

- Without salt: 1 billion passwords tested in seconds
- With salt: Must crack each user separately = years

---

### Answer 3: How bcrypt works

**bcrypt = Blowfish + adaptive cost**

**Key features:**

**1. Adaptive cost (work factor)**
```
Cost 10: ~1,000 hashes/second
Cost 12: ~250 hashes/second
Cost 14: ~60 hashes/second
```

**Higher cost = slower hashing = more security**

```go
bcrypt.GenerateFromPassword(password, 10)
//                                   ↑
//                               Cost factor
```

**2. Automatic salt generation**

```go
hash, _ := bcrypt.GenerateFromPassword([]byte("password"), 10)
// bcrypt generates random salt internally
```

**3. Salt included in output**

```
$2a$10$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy
│  │  │   └────────────────────────┬────────────────────────────┘
│  │  │                            └─ Hash (31 chars)
│  │  └─────────────────────────────── Salt (22 chars)
│  └────────────────────────────────── Cost (10 = 2^10 iterations)
└───────────────────────────────────── Algorithm ($2a = bcrypt)
```

**How it works internally:**

```
1. Generate random 16-byte salt
2. Derive key using expensive key derivation function (EKF)
   - Runs 2^cost iterations (2^10 = 1024 iterations)
   - Uses Blowfish cipher
   - Memory-hard operation
3. Encode result (algorithm + cost + salt + hash)
```

**Why it's better:**

**MD5/SHA-256:**
- Single pass
- GPU-friendly
- Billions of hashes/second

**bcrypt:**
- Multiple iterations (2^cost)
- GPU-resistant
- ~1000 hashes/second
- Automatically salted

**Comparison:**

```
Crack 8-character password:

MD5:
- GPU: 2 billion hashes/sec
- Time to crack: 3 hours

bcrypt (cost 10):
- CPU: 1000 hashes/sec
- Time to crack: 60,000 years
```

---

### Answer 4: Implementation

```go
package main

import (
    "database/sql"
    "encoding/json"
    "errors"
    "net/http"
    "time"
    
    "golang.org/x/crypto/bcrypt"
    _ "github.com/mattn/go-sqlite3"
)

type User struct {
    ID           int       `json:"id"`
    Email        string    `json:"email"`
    PasswordHash string    `json:"-"`  // Never expose in JSON
    CreatedAt    time.Time `json:"created_at"`
}

// Initialize database
func InitDB() (*sql.DB, error) {
    db, err := sql.Open("sqlite3", "./users.db")
    if err != nil {
        return nil, err
    }
    
    _, err = db.Exec(`
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            email TEXT UNIQUE NOT NULL,
            password_hash TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    `)
    
    return db, err
}

// Register new user
func RegisterUser(db *sql.DB, email, password string) (*User, error) {
    // Validate email
    if email == "" {
        return nil, errors.New("email is required")
    }
    
    // Validate password strength
    if len(password) < 8 {
        return nil, errors.New("password must be at least 8 characters")
    }
    
    // Hash password (cost 10)
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(password), 10)
    if err != nil {
        return nil, err
    }
    
    // Insert into database
    result, err := db.Exec(
        "INSERT INTO users (email, password_hash) VALUES (?, ?)",
        email,
        string(hashedPassword),
    )
    
    if err != nil {
        // Check for duplicate email
        if err.Error() == "UNIQUE constraint failed: users.email" {
            return nil, errors.New("email already exists")
        }
        return nil, err
    }
    
    // Get inserted ID
    id, _ := result.LastInsertId()
    
    return &User{
        ID:        int(id),
        Email:     email,
        CreatedAt: time.Now(),
    }, nil
}

// Login user
func LoginUser(db *sql.DB, email, password string) (*User, error) {
    var user User
    
    // Get user by email
    err := db.QueryRow(
        "SELECT id, email, password_hash, created_at FROM users WHERE email = ?",
        email,
    ).Scan(&user.ID, &user.Email, &user.PasswordHash, &user.CreatedAt)
    
    if err == sql.ErrNoRows {
        // User not found - use generic error
        // Don't reveal if email exists
        return nil, errors.New("invalid email or password")
    }
    if err != nil {
        return nil, err
    }
    
    // Verify password
    err = bcrypt.CompareHashAndPassword(
        []byte(user.PasswordHash),
        []byte(password),
    )
    
    if err != nil {
        return nil, errors.New("invalid email or password")
    }
    
    return &user, nil
}

// HTTP Handlers

func RegisterHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            Email    string `json:"email"`
            Password string `json:"password"`
        }
        
        // Parse request
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "Invalid request", 400)
            return
        }
        
        // Register user
        user, err := RegisterUser(db, req.Email, req.Password)
        if err != nil {
            http.Error(w, err.Error(), 400)
            return
        }
        
        // Return success
        w.WriteHeader(201)
        json.NewEncoder(w).Encode(map[string]interface{}{
            "message": "User created successfully",
            "user":    user,
        })
    }
}

func LoginHandler(db *sql.DB) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        var req struct {
            Email    string `json:"email"`
            Password string `json:"password"`
        }
        
        // Parse request
        if err := json.NewDecoder(r.Body).Decode(&req); err != nil {
            http.Error(w, "Invalid request", 400)
            return
        }
        
        // Login user
        user, err := LoginUser(db, req.Email, req.Password)
        if err != nil {
            http.Error(w, err.Error(), 401)
            return
        }
        
        // In production: Generate JWT token here
        // token := GenerateJWT(user.ID)
        
        // Return success
        json.NewEncoder(w).Encode(map[string]interface{}{
            "message": "Login successful",
            "user":    user,
            // "token": token,
        })
    }
}

func main() {
    db, err := InitDB()
    if err != nil {
        panic(err)
    }
    defer db.Close()
    
    http.HandleFunc("/register", RegisterHandler(db))
    http.HandleFunc("/login", LoginHandler(db))
    
    http.ListenAndServe(":8080", nil)
}
```

**Usage:**

**Register:**
```bash
curl -X POST http://localhost:8080/register \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"securepass123"}'
```

**Login:**
```bash
curl -X POST http://localhost:8080/login \
  -H "Content-Type: application/json" \
  -d '{"email":"john@example.com","password":"securepass123"}'
```

---

### Answer 5: Password Reset

**Secure password reset flow:**

```
1. User requests reset
   POST /auth/forgot-password
   { "email": "john@example.com" }

2. Server:
   a. Generate secure random token (32+ bytes)
   b. Hash token before storing
   c. Store hashed token + expiry (15-60 min)
   d. Send unhashed token via email

3. User clicks email link
   https://example.com/reset-password?token=xyz123

4. User submits new password
   POST /auth/reset-password
   { "token": "xyz123", "new_password": "..." }

5. Server:
   a. Hash submitted token
   b. Look up hashed token in database
   c. Check if expired
   d. Hash new password
   e. Update user password
   f. Delete reset token
```

**Implementation:**

```go
import (
    "crypto/rand"
    "crypto/sha256"
    "encoding/hex"
    "time"
)

// Generate reset token
func GenerateResetToken() (string, string, error) {
    // Generate 32 random bytes
    bytes := make([]byte, 32)
    _, err := rand.Read(bytes)
    if err != nil {
        return "", "", err
    }
    
    // Hex encode for URL safety
    token := hex.EncodeToString(bytes)
    
    // Hash for database storage
    hash := sha256.Sum256([]byte(token))
    tokenHash := hex.EncodeToString(hash[:])
    
    return token, tokenHash, nil
}

// Request password reset
func RequestPasswordReset(db *sql.DB, email string) error {
    // Check if user exists
    var userID int
    err := db.QueryRow("SELECT id FROM users WHERE email = ?", email).Scan(&userID)
    if err != nil {
        // Don't reveal if email exists
        return nil  // Always return success
    }
    
    // Generate token
    token, tokenHash, err := GenerateResetToken()
    if err != nil {
        return err
    }
    
    // Store hashed token with expiry (15 minutes)
    expiry := time.Now().Add(15 * time.Minute)
    
    _, err = db.Exec(`
        INSERT INTO password_resets (user_id, token_hash, expires_at)
        VALUES (?, ?, ?)
        ON CONFLICT(user_id) DO UPDATE SET
            token_hash = excluded.token_hash,
            expires_at = excluded.expires_at
    `, userID, tokenHash, expiry)
    
    if err != nil {
        return err
    }
    
    // Send email with unhashed token
    SendResetEmail(email, token)
    
    return nil
}

// Reset password
func ResetPassword(db *sql.DB, token, newPassword string) error {
    // Hash submitted token
    hash := sha256.Sum256([]byte(token))
    tokenHash := hex.EncodeToString(hash[:])
    
    // Find valid token
    var userID int
    var expiresAt time.Time
    
    err := db.QueryRow(`
        SELECT user_id, expires_at
        FROM password_resets
        WHERE token_hash = ?
    `, tokenHash).Scan(&userID, &expiresAt)
    
    if err != nil {
        return errors.New("invalid or expired token")
    }
    
    // Check if expired
    if time.Now().After(expiresAt) {
        return errors.New("invalid or expired token")
    }
    
    // Hash new password
    hashedPassword, err := bcrypt.GenerateFromPassword([]byte(newPassword), 10)
    if err != nil {
        return err
    }
    
    // Update password
    _, err = db.Exec(
        "UPDATE users SET password_hash = ? WHERE id = ?",
        string(hashedPassword),
        userID,
    )
    if err != nil {
        return err
    }
    
    // Delete used token
    db.Exec("DELETE FROM password_resets WHERE user_id = ?", userID)
    
    return nil
}
```

**Security considerations:**

1. ✅ **Token is random and unpredictable** (crypto/rand)
2. ✅ **Hash token before storing** (prevent token theft if DB leaks)
3. ✅ **Short expiry** (15-60 minutes)
4. ✅ **One-time use** (delete after use)
5. ✅ **Don't reveal if email exists** (prevent email enumeration)
6. ✅ **Rate limit** (prevent spam)
7. ✅ **Send to registered email only**

</details>

</details>