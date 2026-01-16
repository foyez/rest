# 4.2 Filtering & Sorting

## Filtering

### Basic Filtering

**Query parameters for filtering:**

```http
GET /products?category=electronics
GET /products?category=electronics&in_stock=true
GET /products?price_min=100&price_max=500
```

**Implementation (Go):**

```go
func GetProducts(w http.ResponseWriter, r *http.Request) {
    query := "SELECT * FROM products WHERE 1=1"
    args := []interface{}{}
    
    // Category filter
    if category := r.URL.Query().Get("category"); category != "" {
        query += " AND category = ?"
        args = append(args, category)
    }
    
    // In stock filter
    if inStock := r.URL.Query().Get("in_stock"); inStock == "true" {
        query += " AND stock > 0"
    }
    
    // Price range
    if priceMin := r.URL.Query().Get("price_min"); priceMin != "" {
        query += " AND price >= ?"
        args = append(args, priceMin)
    }
    
    if priceMax := r.URL.Query().Get("price_max"); priceMax != "" {
        query += " AND price <= ?"
        args = append(args, priceMax)
    }
    
    rows, _ := db.Query(query, args...)
    // Process results...
}
```

---

### Multiple Values (OR logic)

```http
GET /products?category=electronics,books,toys
GET /products?tags=laptop&tags=gaming
```

**Implementation:**

```go
// Option 1: Comma-separated
categories := strings.Split(r.URL.Query().Get("category"), ",")
placeholders := strings.Repeat("?,", len(categories))
placeholders = placeholders[:len(placeholders)-1]

query := fmt.Sprintf("SELECT * FROM products WHERE category IN (%s)", placeholders)

// Option 2: Multiple parameters
categories := r.URL.Query()["category"]
```

---

### Date Filtering

```http
GET /orders?created_after=2025-01-01
GET /orders?created_before=2025-12-31
GET /orders?created_after=2025-01-01&created_before=2025-01-31
```

**Implementation:**

```go
if createdAfter := r.URL.Query().Get("created_after"); createdAfter != "" {
    query += " AND created_at >= ?"
    args = append(args, createdAfter)
}

if createdBefore := r.URL.Query().Get("created_before"); createdBefore != "" {
    query += " AND created_at <= ?"
    args = append(args, createdBefore)
}
```

---

### Search/Text Filtering

```http
GET /products?q=laptop
GET /users?search=john
```

**Implementation:**

```go
if search := r.URL.Query().Get("q"); search != "" {
    query += " AND (name LIKE ? OR description LIKE ?)"
    searchTerm := "%" + search + "%"
    args = append(args, searchTerm, searchTerm)
}
```

---

### Complex Filters (JSON)

For very complex filters, use POST with JSON body:

```http
POST /products/search
{
  "filters": {
    "category": ["electronics", "computers"],
    "price": {
      "min": 100,
      "max": 1000
    },
    "tags": {
      "all": ["laptop", "gaming"]
    }
  }
}
```

---

## Sorting

### Single Field Sort

```http
GET /products?sort=price          # Ascending
GET /products?sort=-price         # Descending (minus prefix)
GET /products?order_by=price&order=asc
GET /products?order_by=price&order=desc
```

**Implementation:**

```go
func GetProducts(w http.ResponseWriter, r *http.Request) {
    query := "SELECT * FROM products"
    
    sortField := r.URL.Query().Get("sort")
    if sortField == "" {
        sortField = "id"  // Default
    }
    
    sortOrder := "ASC"
    if strings.HasPrefix(sortField, "-") {
        sortOrder = "DESC"
        sortField = sortField[1:]  // Remove minus
    }
    
    // Whitelist allowed fields (security!)
    allowedFields := map[string]bool{
        "id": true, "name": true, "price": true, "created_at": true,
    }
    
    if !allowedFields[sortField] {
        http.Error(w, "Invalid sort field", 400)
        return
    }
    
    query += fmt.Sprintf(" ORDER BY %s %s", sortField, sortOrder)
    
    rows, _ := db.Query(query)
    // Process results...
}
```

---

### Multiple Field Sort

```http
GET /products?sort=category,-price,name
```

**Meaning:** 
1. Sort by category (ascending)
2. Then by price (descending)
3. Then by name (ascending)

**Implementation:**

```go
sortFields := strings.Split(r.URL.Query().Get("sort"), ",")
orderClauses := []string{}

for _, field := range sortFields {
    order := "ASC"
    if strings.HasPrefix(field, "-") {
        order = "DESC"
        field = field[1:]
    }
    
    if !allowedFields[field] {
        http.Error(w, "Invalid sort field: "+field, 400)
        return
    }
    
    orderClauses = append(orderClauses, fmt.Sprintf("%s %s", field, order))
}

query += " ORDER BY " + strings.Join(orderClauses, ", ")
```

**SQL output:**
```sql
ORDER BY category ASC, price DESC, name ASC
```

---

## Filter Builder Pattern

```go
type FilterBuilder struct {
    query string
    args  []interface{}
}

func NewFilterBuilder() *FilterBuilder {
    return &FilterBuilder{
        query: "SELECT * FROM products WHERE 1=1",
        args:  []interface{}{},
    }
}

func (fb *FilterBuilder) AddFilter(field, operator string, value interface{}) *FilterBuilder {
    fb.query += fmt.Sprintf(" AND %s %s ?", field, operator)
    fb.args = append(fb.args, value)
    return fb
}

func (fb *FilterBuilder) AddSort(field, order string) *FilterBuilder {
    fb.query += fmt.Sprintf(" ORDER BY %s %s", field, order)
    return fb
}

func (fb *FilterBuilder) Build() (string, []interface{}) {
    return fb.query, fb.args
}

// Usage
func GetProducts(w http.ResponseWriter, r *http.Request) {
    fb := NewFilterBuilder()
    
    if category := r.URL.Query().Get("category"); category != "" {
        fb.AddFilter("category", "=", category)
    }
    
    if priceMin := r.URL.Query().Get("price_min"); priceMin != "" {
        fb.AddFilter("price", ">=", priceMin)
    }
    
    if sort := r.URL.Query().Get("sort"); sort != "" {
        order := "ASC"
        if strings.HasPrefix(sort, "-") {
            order = "DESC"
            sort = sort[1:]
        }
        fb.AddSort(sort, order)
    }
    
    query, args := fb.Build()
    rows, _ := db.Query(query, args...)
}
```

---

## Security Considerations

### 1. SQL Injection Prevention

**❌ Vulnerable:**
```go
query := fmt.Sprintf("SELECT * FROM products WHERE category = '%s'", category)
db.Query(query)  // SQL injection!
```

**✅ Safe:**
```go
query := "SELECT * FROM products WHERE category = ?"
db.Query(query, category)  // Parameterized
```

---

### 2. Field Whitelisting

**❌ Dangerous:**
```go
sortField := r.URL.Query().Get("sort")
query := fmt.Sprintf("ORDER BY %s", sortField)
// Attacker: ?sort=1; DROP TABLE products--
```

**✅ Safe:**
```go
allowedFields := map[string]bool{
    "id": true, "name": true, "price": true,
}

if !allowedFields[sortField] {
    return errors.New("invalid field")
}
```

---

### 3. Limit Number of Filters

```go
const MaxFilters = 10

filters := parseFilters(r)
if len(filters) > MaxFilters {
    http.Error(w, "Too many filters", 400)
    return
}
```

---

## Advanced Filtering

### Range Queries

```http
GET /products?price=100..500
GET /orders?created_at=2025-01-01..2025-01-31
```

**Implementation:**

```go
if priceRange := r.URL.Query().Get("price"); strings.Contains(priceRange, "..") {
    parts := strings.Split(priceRange, "..")
    if len(parts) == 2 {
        query += " AND price BETWEEN ? AND ?"
        args = append(args, parts[0], parts[1])
    }
}
```

---

### Negation

```http
GET /products?category!=electronics
GET /users?status!=deleted
```

**Implementation:**

```go
if category := r.URL.Query().Get("category!"); category != "" {
    query += " AND category != ?"
    args = append(args, category)
}
```

---

### Null Checks

```http
GET /products?discount=null
GET /products?discount!=null
```

**Implementation:**

```go
if discount := r.URL.Query().Get("discount"); discount == "null" {
    query += " AND discount IS NULL"
} else if discount := r.URL.Query().Get("discount!"); discount == "null" {
    query += " AND discount IS NOT NULL"
}
```

---

## Best Practices

### 1. Document Filter Parameters

```yaml
# OpenAPI spec
parameters:
  - name: category
    in: query
    schema:
      type: string
      enum: [electronics, books, clothing]
  - name: price_min
    in: query
    schema:
      type: number
  - name: sort
    in: query
    schema:
      type: string
      pattern: "^-?(id|name|price|created_at)$"
```

---

### 2. Provide Defaults

```go
sortField := r.URL.Query().Get("sort")
if sortField == "" {
    sortField = "created_at"  // Default sort
}

sortOrder := r.URL.Query().Get("order")
if sortOrder == "" {
    sortOrder = "DESC"  // Default order
}
```

---

### 3. Return Filter Metadata

```json
{
  "data": [...],
  "filters_applied": {
    "category": "electronics",
    "price_min": 100,
    "sort": "-price"
  },
  "available_filters": {
    "category": ["electronics", "books", "clothing"],
    "in_stock": [true, false]
  }
}
```

---

### 4. Validate Filter Values

```go
func validateCategory(category string) error {
    validCategories := []string{"electronics", "books", "clothing"}
    
    for _, valid := range validCategories {
        if category == valid {
            return nil
        }
    }
    
    return errors.New("invalid category")
}
```

---

## Complete Example

```go
package main

import (
    "database/sql"
    "encoding/json"
    "fmt"
    "net/http"
    "strconv"
    "strings"
)

type Product struct {
    ID          int     `json:"id"`
    Name        string  `json:"name"`
    Category    string  `json:"category"`
    Price       float64 `json:"price"`
    InStock     bool    `json:"in_stock"`
    CreatedAt   string  `json:"created_at"`
}

type ProductsResponse struct {
    Data    []Product              `json:"data"`
    Filters map[string]interface{} `json:"filters_applied"`
    Total   int                    `json:"total"`
}

var allowedSortFields = map[string]bool{
    "id": true, "name": true, "price": true, "created_at": true,
}

var allowedCategories = map[string]bool{
    "electronics": true, "books": true, "clothing": true,
}

func GetProducts(w http.ResponseWriter, r *http.Request, db *sql.DB) {
    query := "SELECT id, name, category, price, in_stock, created_at FROM products WHERE 1=1"
    args := []interface{}{}
    filters := make(map[string]interface{})
    
    // Category filter
    if category := r.URL.Query().Get("category"); category != "" {
        if !allowedCategories[category] {
            http.Error(w, "Invalid category", 400)
            return
        }
        query += " AND category = ?"
        args = append(args, category)
        filters["category"] = category
    }
    
    // Price range
    if priceMin := r.URL.Query().Get("price_min"); priceMin != "" {
        if price, err := strconv.ParseFloat(priceMin, 64); err == nil {
            query += " AND price >= ?"
            args = append(args, price)
            filters["price_min"] = price
        }
    }
    
    if priceMax := r.URL.Query().Get("price_max"); priceMax != "" {
        if price, err := strconv.ParseFloat(priceMax, 64); err == nil {
            query += " AND price <= ?"
            args = append(args, price)
            filters["price_max"] = price
        }
    }
    
    // In stock filter
    if inStock := r.URL.Query().Get("in_stock"); inStock != "" {
        query += " AND in_stock = ?"
        args = append(args, inStock == "true")
        filters["in_stock"] = inStock == "true"
    }
    
    // Search
    if search := r.URL.Query().Get("q"); search != "" {
        query += " AND (name LIKE ? OR category LIKE ?)"
        searchTerm := "%" + search + "%"
        args = append(args, searchTerm, searchTerm)
        filters["q"] = search
    }
    
    // Get total count before pagination
    countQuery := strings.Replace(query, "SELECT id, name, category, price, in_stock, created_at", "SELECT COUNT(*)", 1)
    var total int
    db.QueryRow(countQuery, args...).Scan(&total)
    
    // Sorting
    sortField := r.URL.Query().Get("sort")
    if sortField == "" {
        sortField = "id"
    }
    
    sortOrder := "ASC"
    if strings.HasPrefix(sortField, "-") {
        sortOrder = "DESC"
        sortField = sortField[1:]
    }
    
    if !allowedSortFields[sortField] {
        http.Error(w, "Invalid sort field", 400)
        return
    }
    
    query += fmt.Sprintf(" ORDER BY %s %s", sortField, sortOrder)
    filters["sort"] = r.URL.Query().Get("sort")
    
    // Pagination
    limit := 20
    if l := r.URL.Query().Get("limit"); l != "" {
        if parsedLimit, err := strconv.Atoi(l); err == nil && parsedLimit > 0 {
            if parsedLimit > 100 {
                parsedLimit = 100
            }
            limit = parsedLimit
        }
    }
    
    offset := 0
    if page := r.URL.Query().Get("page"); page != "" {
        if p, err := strconv.Atoi(page); err == nil && p > 0 {
            offset = (p - 1) * limit
        }
    }
    
    query += " LIMIT ? OFFSET ?"
    args = append(args, limit, offset)
    
    // Execute query
    rows, err := db.Query(query, args...)
    if err != nil {
        http.Error(w, err.Error(), 500)
        return
    }
    defer rows.Close()
    
    products := []Product{}
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name, &p.Category, &p.Price, &p.InStock, &p.CreatedAt)
        products = append(products, p)
    }
    
    response := ProductsResponse{
        Data:    products,
        Filters: filters,
        Total:   total,
    }
    
    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(response)
}
```

**Usage:**
```bash
curl "http://localhost:8080/products?category=electronics&price_min=100&price_max=500&sort=-price&page=1&limit=20"
```

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Filters should use __________ parameters in REST APIs.
2. To sort descending, use a __________ prefix before the field name.
3. Always __________ sort field names to prevent SQL injection.
4. For complex filters, use __________ with JSON body instead of GET.

### True/False

1. ⬜ Sort fields should be directly concatenated into SQL queries
2. ⬜ Multiple filters should use AND logic by default
3. ⬜ Allowing unlimited filters is a security risk
4. ⬜ Parameterized queries prevent SQL injection
5. ⬜ Case-insensitive search requires LIKE operator

### Multiple Choice

1. Which is the correct way to filter by category?
   - A) `/products/electronics`
   - B) `/products?category=electronics`
   - C) `/products?filter=category:electronics`
   - D) `POST /products/filter`

2. How to sort by price descending?
   - A) `?sort=price&desc=true`
   - B) `?sort=-price`
   - C) `?sort=price&order=desc`
   - D) Both B and C

3. What's the safest way to build SQL queries?
   - A) String concatenation
   - B) fmt.Sprintf
   - C) Parameterized queries
   - D) User input directly

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. query
2. minus (or -)
3. whitelist
4. POST

**True/False:**
1. ❌ False - Must whitelist fields for security
2. ✅ True - Multiple filters typically use AND
3. ✅ True - Set maximum number of filters
4. ✅ True - Parameterized queries are safe
5. ✅ True - LIKE with % wildcards for searching

**Multiple Choice:**
1. **B** - Query parameters are standard for filtering
2. **D** - Both `-` prefix and explicit `order=desc` work
3. **C** - Parameterized queries prevent SQL injection

</details>

</details>