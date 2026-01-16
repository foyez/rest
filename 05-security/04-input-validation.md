# 5.4 Input Validation

## Why Validate Input?

**Never trust user input!**

```
User input → Validation → Process

Without validation:
- SQL injection
- XSS attacks
- Buffer overflow
- Business logic errors
- Data corruption
```

---

## Validation Principles

### 1. Whitelist, Not Blacklist

```go
❌ Blacklist (reject bad):
if input contains "script" or "DROP" or "..." {
    reject
}
// Infinite bad patterns

✅ Whitelist (accept good):
if input matches [a-zA-Z0-9] {
    accept
}
// Finite good patterns
```

---

### 2. Validate Early

```
Client → Validation → Business Logic → Database

Fail fast!
```

---

### 3. Validate on Server

```
❌ Client-side only (can be bypassed)
✅ Server-side always
✅ Client + Server (best UX)
```

---

## Types of Validation

### 1. Type Validation

```go
// Ensure correct type
var req struct {
    Age int `json:"age"`  // Must be integer
}

json.NewDecoder(r.Body).Decode(&req)
// Fails if age is "abc"
```

---

### 2. Format Validation

**Email:**
```go
import "regexp"

func ValidateEmail(email string) bool {
    pattern := `^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$`
    matched, _ := regexp.MatchString(pattern, email)
    return matched
}
```

**Phone:**
```go
func ValidatePhone(phone string) bool {
    // US format: (123) 456-7890 or 123-456-7890
    pattern := `^\(?\d{3}\)?[-.\s]?\d{3}[-.\s]?\d{4}$`
    matched, _ := regexp.MatchString(pattern, phone)
    return matched
}
```

**URL:**
```go
import "net/url"

func ValidateURL(urlStr string) bool {
    u, err := url.Parse(urlStr)
    return err == nil && u.Scheme != "" && u.Host != ""
}
```

---

### 3. Range Validation

```go
func ValidateAge(age int) error {
    if age < 0 {
        return errors.New("age cannot be negative")
    }
    if age > 150 {
        return errors.New("age too high")
    }
    return nil
}

func ValidatePassword(password string) error {
    if len(password) < 8 {
        return errors.New("password must be at least 8 characters")
    }
    if len(password) > 128 {
        return errors.New("password too long")
    }
    return nil
}
```

---

### 4. Length Validation

```go
func ValidateUsername(username string) error {
    if len(username) < 3 {
        return errors.New("username too short")
    }
    if len(username) > 20 {
        return errors.New("username too long")
    }
    return nil
}
```

---

### 5. Pattern Validation

```go
func ValidateUsername(username string) error {
    // Only alphanumeric and underscore
    pattern := `^[a-zA-Z0-9_]+$`
    matched, _ := regexp.MatchString(pattern, username)
    
    if !matched {
        return errors.New("username can only contain letters, numbers, and underscore")
    }
    
    return nil
}
```

---

### 6. Required Fields

```go
type User struct {
    Name  string `json:"name" validate:"required"`
    Email string `json:"email" validate:"required,email"`
    Age   int    `json:"age" validate:"required,min=18,max=120"`
}

func ValidateUser(user *User) error {
    if user.Name == "" {
        return errors.New("name is required")
    }
    if user.Email == "" {
        return errors.New("email is required")
    }
    return nil
}
```

---

## Validation Library (go-playground/validator)

```go
import "github.com/go-playground/validator/v10"

type User struct {
    Name     string `validate:"required,min=3,max=50"`
    Email    string `validate:"required,email"`
    Age      int    `validate:"required,gte=18,lte=120"`
    Password string `validate:"required,min=8"`
    Website  string `validate:"omitempty,url"`
}

func ValidateStruct(user *User) error {
    validate := validator.New()
    err := validate.Struct(user)
    
    if err != nil {
        // Return validation errors
        for _, err := range err.(validator.ValidationErrors) {
            fmt.Printf("Field: %s, Error: %s\n", err.Field(), err.Tag())
        }
        return err
    }
    
    return nil
}
```

**Common validation tags:**
```
required         - Field must not be empty
email            - Valid email format
url              - Valid URL
min=3            - Minimum value/length
max=100          - Maximum value/length
gte=18           - Greater than or equal
lte=120          - Less than or equal
len=10           - Exact length
alpha            - Only letters
alphanum         - Letters and numbers
numeric          - Only numbers
```

---

## Custom Validators

```go
import "github.com/go-playground/validator/v10"

// Register custom validator
func RegisterValidators(v *validator.Validate) {
    v.RegisterValidation("username", validateUsername)
    v.RegisterValidation("strong_password", validateStrongPassword)
}

// Custom username validator
func validateUsername(fl validator.FieldLevel) bool {
    username := fl.Field().String()
    
    // 3-20 characters, alphanumeric and underscore only
    if len(username) < 3 || len(username) > 20 {
        return false
    }
    
    pattern := `^[a-zA-Z0-9_]+$`
    matched, _ := regexp.MatchString(pattern, username)
    
    return matched
}

// Custom strong password validator
func validateStrongPassword(fl validator.FieldLevel) bool {
    password := fl.Field().String()
    
    // At least 8 chars, 1 uppercase, 1 lowercase, 1 number
    if len(password) < 8 {
        return false
    }
    
    hasUpper := regexp.MustCompile(`[A-Z]`).MatchString(password)
    hasLower := regexp.MustCompile(`[a-z]`).MatchString(password)
    hasNumber := regexp.MustCompile(`[0-9]`).MatchString(password)
    
    return hasUpper && hasLower && hasNumber
}

// Usage
type User struct {
    Username string `validate:"required,username"`
    Password string `validate:"required,strong_password"`
}
```

---

## Sanitization vs Validation

**Validation** = Check if input is acceptable
**Sanitization** = Clean/modify input to make it safe

### Sanitization Examples

```go
import (
    "html"
    "strings"
)

// Remove HTML tags
func SanitizeHTML(input string) string {
    return html.EscapeString(input)
}

// Trim whitespace
func SanitizeString(input string) string {
    return strings.TrimSpace(input)
}

// Remove special characters
func SanitizeAlphanumeric(input string) string {
    re := regexp.MustCompile(`[^a-zA-Z0-9]`)
    return re.ReplaceAllString(input, "")
}

// Normalize email
func NormalizeEmail(email string) string {
    email = strings.TrimSpace(email)
    email = strings.ToLower(email)
    return email
}
```

---

## Validation Middleware

```go
func ValidateUserRequest(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var user User
        
        // Decode request
        err := json.NewDecoder(r.Body).Decode(&user)
        if err != nil {
            http.Error(w, "Invalid JSON", 400)
            return
        }
        
        // Validate
        validate := validator.New()
        err = validate.Struct(user)
        if err != nil {
            errors := make(map[string]string)
            for _, err := range err.(validator.ValidationErrors) {
                errors[err.Field()] = getErrorMessage(err)
            }
            
            w.WriteHeader(422)
            json.NewEncoder(w).Encode(map[string]interface{}{
                "error":   "validation_failed",
                "details": errors,
            })
            return
        }
        
        // Add validated user to context
        ctx := context.WithValue(r.Context(), "user", user)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

func getErrorMessage(err validator.FieldError) string {
    switch err.Tag() {
    case "required":
        return err.Field() + " is required"
    case "email":
        return "Invalid email format"
    case "min":
        return err.Field() + " must be at least " + err.Param()
    case "max":
        return err.Field() + " must be at most " + err.Param()
    default:
        return "Invalid " + err.Field()
    }
}
```

---

## Common Validation Scenarios

### 1. User Registration

```go
type RegisterRequest struct {
    Username string `validate:"required,min=3,max=20,alphanum"`
    Email    string `validate:"required,email"`
    Password string `validate:"required,min=8,max=128"`
    Age      int    `validate:"required,gte=18"`
}

func RegisterHandler(w http.ResponseWriter, r *http.Request) {
    var req RegisterRequest
    json.NewDecoder(r.Body).Decode(&req)
    
    // Validate
    validate := validator.New()
    if err := validate.Struct(req); err != nil {
        // Return errors
        return
    }
    
    // Additional business validation
    if UserExists(req.Email) {
        http.Error(w, "Email already exists", 409)
        return
    }
    
    // Create user...
}
```

---

### 2. Credit Card

```go
func ValidateCreditCard(number string) bool {
    // Remove spaces/dashes
    number = strings.ReplaceAll(number, " ", "")
    number = strings.ReplaceAll(number, "-", "")
    
    // Length check
    if len(number) < 13 || len(number) > 19 {
        return false
    }
    
    // Luhn algorithm
    return luhnCheck(number)
}

func luhnCheck(number string) bool {
    sum := 0
    alternate := false
    
    for i := len(number) - 1; i >= 0; i-- {
        digit := int(number[i] - '0')
        
        if alternate {
            digit *= 2
            if digit > 9 {
                digit -= 9
            }
        }
        
        sum += digit
        alternate = !alternate
    }
    
    return sum%10 == 0
}
```

---

### 3. File Upload

```go
func ValidateFileUpload(file multipart.File, header *multipart.FileHeader) error {
    // Check size (10MB max)
    if header.Size > 10*1024*1024 {
        return errors.New("file too large (max 10MB)")
    }
    
    // Check file type
    allowedTypes := map[string]bool{
        "image/jpeg": true,
        "image/png":  true,
        "image/gif":  true,
    }
    
    contentType := header.Header.Get("Content-Type")
    if !allowedTypes[contentType] {
        return errors.New("invalid file type")
    }
    
    // Check magic bytes (first few bytes)
    buffer := make([]byte, 512)
    _, err := file.Read(buffer)
    if err != nil {
        return err
    }
    
    detectedType := http.DetectContentType(buffer)
    if !allowedTypes[detectedType] {
        return errors.New("file type mismatch")
    }
    
    // Reset file pointer
    file.Seek(0, 0)
    
    return nil
}
```

---

### 4. Date Validation

```go
func ValidateDate(dateStr string) error {
    // Parse ISO 8601 format
    _, err := time.Parse("2006-01-02", dateStr)
    if err != nil {
        return errors.New("invalid date format (use YYYY-MM-DD)")
    }
    return nil
}

func ValidateDateRange(start, end string) error {
    startDate, _ := time.Parse("2006-01-02", start)
    endDate, _ := time.Parse("2006-01-02", end)
    
    if endDate.Before(startDate) {
        return errors.New("end date must be after start date")
    }
    
    return nil
}
```

---

## Security Validation

### 1. Prevent XSS

```go
import "html"

// Escape HTML
func SanitizeHTML(input string) string {
    return html.EscapeString(input)
}

// Example:
// Input:  <script>alert('XSS')</script>
// Output: &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;
```

---

### 2. Prevent Path Traversal

```go
import "path/filepath"

func ValidateFilePath(filename string) error {
    // Prevent ../../../etc/passwd
    if strings.Contains(filename, "..") {
        return errors.New("invalid filename")
    }
    
    // Clean path
    cleaned := filepath.Clean(filename)
    if cleaned != filename {
        return errors.New("invalid filename")
    }
    
    return nil
}
```

---

### 3. Prevent Command Injection

```go
func ValidateCommand(input string) error {
    // Whitelist characters
    pattern := `^[a-zA-Z0-9_\-]+$`
    matched, _ := regexp.MatchString(pattern, input)
    
    if !matched {
        return errors.New("invalid characters in command")
    }
    
    // Blacklist dangerous commands
    dangerous := []string{"rm", "dd", "mkfs", "shutdown"}
    for _, cmd := range dangerous {
        if strings.Contains(input, cmd) {
            return errors.New("dangerous command detected")
        }
    }
    
    return nil
}
```

---

## Best Practices

### 1. Fail Fast

```go
// Validate immediately
if err := ValidateInput(input); err != nil {
    return err  // Don't process invalid data
}
```

---

### 2. Clear Error Messages

```go
✅ "Email is required"
✅ "Password must be at least 8 characters"
❌ "Invalid input"
❌ "Error 422"
```

---

### 3. Validate on Multiple Layers

```
1. Client-side (UX)
2. API layer (Security)
3. Business logic (Integrity)
4. Database constraints (Final safety)
```

---

### 4. Use Strong Types

```go
type Email string
type Age int

func NewEmail(s string) (Email, error) {
    if !ValidateEmail(s) {
        return "", errors.New("invalid email")
    }
    return Email(s), nil
}
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Input validation should use __________ approach, not blacklist.
2. Validation should happen on the __________ side for security.
3. __________ cleans input to make it safe, while validation checks if acceptable.
4. File uploads should validate both __________ and magic bytes.

### True/False

1. ⬜ Client-side validation is sufficient for security
2. ⬜ You should sanitize all user input
3. ⬜ Validation errors should return 400 Bad Request
4. ⬜ Email validation guarantees the email exists
5. ⬜ Whitelist validation is safer than blacklist

### Multiple Choice

1. Which validation approach is more secure?
   - A) Blacklist bad patterns
   - B) Whitelist good patterns
   - C) No validation
   - D) Both equal

2. What status code for validation errors?
   - A) 400
   - B) 401
   - C) 422
   - D) 500

3. How to prevent XSS in user input?
   - A) Strip all HTML
   - B) Escape HTML entities
   - C) Block script tags
   - D) Use HTTPS

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. whitelist
2. server
3. Sanitization
4. MIME type (or Content-Type)

**True/False:**
1. ❌ False - Client-side can be bypassed; server-side required
2. ✅ True - Always sanitize to prevent injection attacks
3. ❌ False - Use 422 Unprocessable Entity for validation
4. ❌ False - Only validates format, not existence
5. ✅ True - Whitelist is more secure (finite good patterns)

**Multiple Choice:**
1. **B** - Whitelist (accept known good patterns)
2. **C** - 422 Unprocessable Entity
3. **B** - Escape HTML entities (preserve content safely)

</details>

</details>