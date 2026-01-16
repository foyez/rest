# 5.5 SQL Injection Prevention

## What is SQL Injection?

**SQL Injection** = Attacker injects malicious SQL code through user input

**Example attack:**
```sql
-- Normal query
SELECT * FROM users WHERE username = 'john'

-- Attacker input: admin' --
SELECT * FROM users WHERE username = 'admin' --'
                                            └─ Comment, ignores password check

-- Result: Attacker logs in as admin!
```

---

## How SQL Injection Works

### Example 1: Authentication Bypass

**Vulnerable code:**
```go
username := r.FormValue("username")
password := r.FormValue("password")

query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s' AND password = '%s'", 
    username, password)

rows, _ := db.Query(query)
```

**Attack:**
```
Username: admin' --
Password: anything

Generated SQL:
SELECT * FROM users WHERE username = 'admin' --' AND password = 'anything'

Everything after -- is commented out!
Result: Login without password
```

---

### Example 2: Data Theft

**Vulnerable code:**
```go
id := r.URL.Query().Get("id")
query := fmt.Sprintf("SELECT * FROM users WHERE id = %s", id)
```

**Attack:**
```
id=1 UNION SELECT credit_card, ssn, password FROM sensitive_data

Generated SQL:
SELECT * FROM users WHERE id = 1 
UNION SELECT credit_card, ssn, password FROM sensitive_data

Result: Attacker gets sensitive data!
```

---

### Example 3: Database Destruction

**Attack:**
```
id=1; DROP TABLE users; --

Generated SQL:
SELECT * FROM users WHERE id = 1; 
DROP TABLE users; 
--

Result: All user data deleted!
```

---

## Prevention: Parameterized Queries

### ❌ Vulnerable (String Concatenation)

```go
// NEVER DO THIS
query := fmt.Sprintf("SELECT * FROM users WHERE username = '%s'", username)
db.Query(query)
```

---

### ✅ Safe (Parameterized)

```go
// ALWAYS DO THIS
query := "SELECT * FROM users WHERE username = ?"
db.Query(query, username)
```

**How it works:**
```
Database separates:
1. SQL structure:  SELECT * FROM users WHERE username = ?
2. Data:           "admin' --"

Data is treated as data, never as SQL code!
```

---

## Parameterized Queries Examples

### Go (database/sql)

```go
// Single parameter
query := "SELECT * FROM users WHERE id = ?"
row := db.QueryRow(query, userID)

// Multiple parameters
query := "SELECT * FROM users WHERE username = ? AND email = ?"
rows, err := db.Query(query, username, email)

// INSERT
query := "INSERT INTO users (username, email) VALUES (?, ?)"
result, err := db.Exec(query, username, email)

// UPDATE
query := "UPDATE users SET email = ? WHERE id = ?"
db.Exec(query, newEmail, userID)

// DELETE
query := "DELETE FROM users WHERE id = ?"
db.Exec(query, userID)
```

---

### PostgreSQL ($1, $2)

```go
query := "SELECT * FROM users WHERE username = $1 AND email = $2"
rows, err := db.Query(query, username, email)
```

---

### Named Parameters

```go
import "github.com/jmoiron/sqlx"

query := "SELECT * FROM users WHERE username = :username AND email = :email"

params := map[string]interface{}{
    "username": username,
    "email":    email,
}

rows, err := db.NamedQuery(query, params)
```

---

## IN Clause (Multiple Values)

### ❌ Vulnerable

```go
ids := []string{"1", "2", "3"}
query := fmt.Sprintf("SELECT * FROM users WHERE id IN (%s)", 
    strings.Join(ids, ","))
```

---

### ✅ Safe (Method 1: Multiple Placeholders)

```go
ids := []int{1, 2, 3}

// Generate placeholders: ?, ?, ?
placeholders := make([]string, len(ids))
args := make([]interface{}, len(ids))

for i, id := range ids {
    placeholders[i] = "?"
    args[i] = id
}

query := fmt.Sprintf("SELECT * FROM users WHERE id IN (%s)", 
    strings.Join(placeholders, ","))

rows, err := db.Query(query, args...)
```

---

### ✅ Safe (Method 2: sqlx.In)

```go
import "github.com/jmoiron/sqlx"

ids := []int{1, 2, 3}

query, args, err := sqlx.In("SELECT * FROM users WHERE id IN (?)", ids)
query = db.Rebind(query)  // Change ? to $1, $2 for PostgreSQL

rows, err := db.Query(query, args...)
```

---

## Dynamic Table/Column Names

**Problem:** You can't parameterize table/column names

```go
❌ This doesn't work:
query := "SELECT * FROM ?"
db.Query(query, tableName)  // Doesn't parameterize table names
```

---

### Solution: Whitelist

```go
func GetFromTable(tableName string) error {
    // Whitelist allowed tables
    allowedTables := map[string]bool{
        "users":    true,
        "products": true,
        "orders":   true,
    }
    
    if !allowedTables[tableName] {
        return errors.New("invalid table name")
    }
    
    // Safe to use in query
    query := fmt.Sprintf("SELECT * FROM %s", tableName)
    rows, err := db.Query(query)
    
    return err
}
```

---

### Column Names (Whitelist)

```go
func GetUsersSorted(sortColumn string) error {
    // Whitelist columns
    allowedColumns := map[string]bool{
        "id":         true,
        "username":   true,
        "email":      true,
        "created_at": true,
    }
    
    if !allowedColumns[sortColumn] {
        return errors.New("invalid sort column")
    }
    
    query := fmt.Sprintf("SELECT * FROM users ORDER BY %s", sortColumn)
    rows, err := db.Query(query)
    
    return err
}
```

---

## LIKE Queries

### ❌ Vulnerable

```go
search := r.URL.Query().Get("search")
query := fmt.Sprintf("SELECT * FROM products WHERE name LIKE '%%%s%%'", search)
```

**Attack:**
```
search=%'; DROP TABLE products; --
```

---

### ✅ Safe

```go
search := r.URL.Query().Get("search")

// Parameterize the value
query := "SELECT * FROM products WHERE name LIKE ?"
searchPattern := "%" + search + "%"

rows, err := db.Query(query, searchPattern)
```

---

## ORM Safety

### GORM (Safe by default)

```go
import "gorm.io/gorm"

// ✅ Safe (parameterized)
db.Where("username = ?", username).First(&user)

// ✅ Safe (struct)
db.Where(&User{Username: username}).First(&user)

// ❌ DANGEROUS (raw SQL)
db.Raw("SELECT * FROM users WHERE username = '" + username + "'").Scan(&user)

// ✅ Safe (parameterized raw SQL)
db.Raw("SELECT * FROM users WHERE username = ?", username).Scan(&user)
```

---

## Testing for SQL Injection

### Common Test Inputs

```
' OR '1'='1
admin' --
1' UNION SELECT NULL, NULL, NULL --
1'; DROP TABLE users; --
' OR 1=1 --
```

### Testing Example

```go
func TestSQLInjection(t *testing.T) {
    testCases := []string{
        "' OR '1'='1",
        "admin' --",
        "1' UNION SELECT NULL --",
    }
    
    for _, input := range testCases {
        // Should not return results or cause errors
        result, err := GetUser(input)
        
        if err == nil && result != nil {
            t.Errorf("SQL injection succeeded with input: %s", input)
        }
    }
}
```

---

## Additional Protection Layers

### 1. Least Privilege

```sql
-- Create user with limited permissions
CREATE USER 'api_user'@'localhost' IDENTIFIED BY 'password';

-- Grant only necessary permissions
GRANT SELECT, INSERT, UPDATE ON mydb.users TO 'api_user'@'localhost';

-- DON'T grant DROP, DELETE, or admin privileges
```

---

### 2. Input Validation

```go
func ValidateUsername(username string) error {
    // Only allow alphanumeric
    pattern := `^[a-zA-Z0-9_]+$`
    matched, _ := regexp.MatchString(pattern, username)
    
    if !matched {
        return errors.New("invalid username format")
    }
    
    return nil
}
```

---

### 3. Escaping (Last Resort)

```go
import "database/sql"

// Only if parameterization is not possible
func EscapeString(input string) string {
    // This is database-specific
    // Better to use parameterized queries!
    return strings.ReplaceAll(input, "'", "''")
}
```

**Note:** Always prefer parameterized queries over escaping!

---

### 4. Web Application Firewall (WAF)

```
Client → WAF → API Server → Database
         └─ Blocks SQL injection patterns
```

Examples:
- AWS WAF
- Cloudflare WAF
- ModSecurity

---

## Complete Safe Example

```go
package main

import (
    "database/sql"
    "encoding/json"
    "net/http"
    "regexp"
    
    _ "github.com/go-sql-driver/mysql"
)

type User struct {
    ID       int    `json:"id"`
    Username string `json:"username"`
    Email    string `json:"email"`
}

var db *sql.DB

// Safe: Get user by ID
func GetUser(w http.ResponseWriter, r *http.Request) {
    id := r.URL.Query().Get("id")
    
    // ✅ Parameterized query
    query := "SELECT id, username, email FROM users WHERE id = ?"
    
    var user User
    err := db.QueryRow(query, id).Scan(&user.ID, &user.Username, &user.Email)
    
    if err == sql.ErrNoRows {
        http.Error(w, "User not found", 404)
        return
    }
    
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    
    json.NewEncoder(w).Encode(user)
}

// Safe: Create user
func CreateUser(w http.ResponseWriter, r *http.Request) {
    var user User
    json.NewDecoder(r.Body).Decode(&user)
    
    // Validate input
    if err := ValidateUsername(user.Username); err != nil {
        http.Error(w, err.Error(), 422)
        return
    }
    
    // ✅ Parameterized query
    query := "INSERT INTO users (username, email) VALUES (?, ?)"
    result, err := db.Exec(query, user.Username, user.Email)
    
    if err != nil {
        http.Error(w, "Failed to create user", 500)
        return
    }
    
    id, _ := result.LastInsertId()
    user.ID = int(id)
    
    json.NewEncoder(w).Encode(user)
}

// Safe: Search users
func SearchUsers(w http.ResponseWriter, r *http.Request) {
    search := r.URL.Query().Get("q")
    
    // ✅ Parameterized LIKE query
    query := "SELECT id, username, email FROM users WHERE username LIKE ?"
    searchPattern := "%" + search + "%"
    
    rows, err := db.Query(query, searchPattern)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    defer rows.Close()
    
    users := []User{}
    for rows.Next() {
        var user User
        rows.Scan(&user.ID, &user.Username, &user.Email)
        users = append(users, user)
    }
    
    json.NewEncoder(w).Encode(users)
}

// Safe: Sort users (with whitelist)
func GetUsersSorted(w http.ResponseWriter, r *http.Request) {
    sortBy := r.URL.Query().Get("sort")
    
    // ✅ Whitelist allowed columns
    allowedColumns := map[string]bool{
        "id":         true,
        "username":   true,
        "created_at": true,
    }
    
    if !allowedColumns[sortBy] {
        sortBy = "id" // Default
    }
    
    // Safe to use in query now
    query := "SELECT id, username, email FROM users ORDER BY " + sortBy
    
    rows, err := db.Query(query)
    if err != nil {
        http.Error(w, "Database error", 500)
        return
    }
    defer rows.Close()
    
    users := []User{}
    for rows.Next() {
        var user User
        rows.Scan(&user.ID, &user.Username, &user.Email)
        users = append(users, user)
    }
    
    json.NewEncoder(w).Encode(users)
}

func ValidateUsername(username string) error {
    pattern := `^[a-zA-Z0-9_]+$`
    matched, _ := regexp.MatchString(pattern, username)
    
    if !matched {
        return errors.New("invalid username")
    }
    
    return nil
}

func main() {
    var err error
    db, err = sql.Open("mysql", "user:password@tcp(localhost:3306)/mydb")
    if err != nil {
        panic(err)
    }
    
    http.HandleFunc("/users", GetUser)
    http.HandleFunc("/users/create", CreateUser)
    http.HandleFunc("/users/search", SearchUsers)
    http.HandleFunc("/users/sorted", GetUsersSorted)
    
    http.ListenAndServe(":8080", nil)
}
```

---

## Detection and Monitoring

### Log Suspicious Patterns

```go
func LogSuspiciousInput(input string) {
    suspicious := []string{
        "' OR '1'='1",
        "UNION SELECT",
        "DROP TABLE",
        "'; --",
    }
    
    for _, pattern := range suspicious {
        if strings.Contains(strings.ToUpper(input), strings.ToUpper(pattern)) {
            log.Warn("Possible SQL injection attempt",
                "input", input,
                "ip", r.RemoteAddr,
            )
            
            // Alert security team
            alertSecurityTeam(input, r.RemoteAddr)
        }
    }
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. SQL injection is prevented using __________ queries.
2. Table and column names should be validated using a __________.
3. Database users should follow the principle of __________ privilege.
4. The placeholder for parameterized queries in PostgreSQL is __________.

### True/False

1. ⬜ String concatenation for SQL queries is safe if input is validated
2. ⬜ Parameterized queries prevent SQL injection
3. ⬜ Table names can be parameterized
4. ⬜ ORMs automatically prevent SQL injection
5. ⬜ Escaping quotes is sufficient to prevent SQL injection

### Multiple Choice

1. Which is vulnerable to SQL injection?
   - A) `db.Query("SELECT * FROM users WHERE id = ?", id)`
   - B) `db.Query(fmt.Sprintf("SELECT * FROM users WHERE id = %s", id))`
   - C) `db.Where("id = ?", id).First(&user)`
   - D) All of the above

2. How to safely handle dynamic table names?
   - A) Parameterize them
   - B) Escape them
   - C) Whitelist them
   - D) Don't use them

3. What's the safest approach for LIKE queries?
   - A) `LIKE '%' + input + '%'`
   - B) `LIKE ?` with pattern in parameter
   - C) Escape special characters
   - D) Use regex instead

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. parameterized (or prepared)
2. whitelist
3. least
4. $1 (or $2, $3, etc.)

**True/False:**
1. ❌ False - Input validation helps but parameterization is required
2. ✅ True - Parameterized queries separate SQL from data
3. ❌ False - Can't parameterize table/column names (use whitelist)
4. ❌ False - Only if used correctly; raw SQL in ORMs can be vulnerable
5. ❌ False - Escaping is error-prone; use parameterization

**Multiple Choice:**
1. **B** - String formatting is vulnerable to injection
2. **C** - Whitelist (can't parameterize table names)
3. **B** - Parameterize the entire pattern

</details>

</details>