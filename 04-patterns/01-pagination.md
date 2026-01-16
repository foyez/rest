# 4.1 Pagination

## Why Paginate?

**Problem:**
```
GET /products
Returns 1,000,000 products
→ 100MB response
→ Client crashes
→ Server overloaded
```

**Solution: Pagination**
```
GET /products?limit=20
Returns 20 products
→ Fast response
→ Better UX
```

---

## Pagination Strategies

### 1. Offset-Based (Page Number) ⭐ Most Common

**How it works:**
```
Page 1: Items 1-20
Page 2: Items 21-40
Page 3: Items 41-60
```

**Request:**
```http
GET /products?page=2&limit=20
```

**Response:**
```json
{
  "data": [
    { "id": 21, "name": "Product 21" },
    { "id": 22, "name": "Product 22" }
  ],
  "pagination": {
    "page": 2,
    "limit": 20,
    "total": 1000,
    "total_pages": 50,
    "has_next": true,
    "has_prev": true
  }
}
```

**SQL:**
```sql
SELECT * FROM products
ORDER BY id
LIMIT 20 OFFSET 20  -- Page 2 (skip first 20)
```

**Pros:**
- ✅ Simple to implement
- ✅ Can jump to any page
- ✅ Shows total count
- ✅ User-friendly (page numbers)

**Cons:**
- ❌ Slow for large offsets
- ❌ Inconsistent if data changes
- ❌ Performance degrades with scale

**Performance issue:**
```sql
-- Page 1 (fast)
LIMIT 20 OFFSET 0

-- Page 1000 (slow!)
LIMIT 20 OFFSET 20000
-- Database scans 20,000 rows to skip them
```

---

### 2. Cursor-Based ⭐ Best for Scale

**How it works:**
```
Cursor = Pointer to specific item
Next page starts AFTER cursor
```

**Request:**
```http
GET /products?limit=20&cursor=eyJpZCI6MjB9
```

**Cursor (base64 encoded):**
```
eyJpZCI6MjB9  →  {"id":20}
```

**Response:**
```json
{
  "data": [
    { "id": 21, "name": "Product 21" },
    { "id": 22, "name": "Product 22" }
  ],
  "pagination": {
    "next_cursor": "eyJpZCI6NDB9",
    "prev_cursor": "eyJpZCI6MH0",
    "has_more": true
  }
}
```

**SQL:**
```sql
SELECT * FROM products
WHERE id > 20  -- After cursor
ORDER BY id
LIMIT 20
```

**Pros:**
- ✅ Consistent results (even if data changes)
- ✅ Fast (indexed WHERE clause)
- ✅ Scales to billions of records
- ✅ No performance degradation

**Cons:**
- ❌ Can't jump to specific page
- ❌ No total count
- ❌ More complex

---

### 3. Keyset Pagination (Similar to Cursor)

**Request:**
```http
GET /products?after_id=20&limit=20
```

**SQL:**
```sql
SELECT * FROM products
WHERE id > 20
ORDER BY id
LIMIT 20
```

**Difference from cursor:** Cursor is encoded, keyset is plain.

---

## Implementation Examples

### Offset-Based (Go)

```go
package main

import (
    "encoding/json"
    "net/http"
    "strconv"
)

type Product struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

type PaginatedResponse struct {
    Data       []Product  `json:"data"`
    Pagination Pagination `json:"pagination"`
}

type Pagination struct {
    Page       int  `json:"page"`
    Limit      int  `json:"limit"`
    Total      int  `json:"total"`
    TotalPages int  `json:"total_pages"`
    HasNext    bool `json:"has_next"`
    HasPrev    bool `json:"has_prev"`
}

func GetProducts(w http.ResponseWriter, r *http.Request) {
    // Parse query parameters
    page, _ := strconv.Atoi(r.URL.Query().Get("page"))
    if page < 1 {
        page = 1
    }
    
    limit, _ := strconv.Atoi(r.URL.Query().Get("limit"))
    if limit < 1 || limit > 100 {
        limit = 20  // Default
    }
    
    // Calculate offset
    offset := (page - 1) * limit
    
    // Get total count
    var total int
    db.QueryRow("SELECT COUNT(*) FROM products").Scan(&total)
    
    // Get products
    rows, _ := db.Query(`
        SELECT id, name FROM products
        ORDER BY id
        LIMIT ? OFFSET ?
    `, limit, offset)
    
    var products []Product
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name)
        products = append(products, p)
    }
    
    // Calculate pagination metadata
    totalPages := (total + limit - 1) / limit
    
    response := PaginatedResponse{
        Data: products,
        Pagination: Pagination{
            Page:       page,
            Limit:      limit,
            Total:      total,
            TotalPages: totalPages,
            HasNext:    page < totalPages,
            HasPrev:    page > 1,
        },
    }
    
    json.NewEncoder(w).Encode(response)
}
```

---

### Cursor-Based (Go)

```go
package main

import (
    "encoding/base64"
    "encoding/json"
    "net/http"
    "strconv"
)

type CursorPaginatedResponse struct {
    Data       []Product         `json:"data"`
    Pagination CursorPagination  `json:"pagination"`
}

type CursorPagination struct {
    NextCursor string `json:"next_cursor,omitempty"`
    PrevCursor string `json:"prev_cursor,omitempty"`
    HasMore    bool   `json:"has_more"`
}

// Encode cursor
func encodeCursor(id int) string {
    cursor := map[string]int{"id": id}
    jsonCursor, _ := json.Marshal(cursor)
    return base64.StdEncoding.EncodeToString(jsonCursor)
}

// Decode cursor
func decodeCursor(cursor string) (int, error) {
    decoded, err := base64.StdEncoding.DecodeString(cursor)
    if err != nil {
        return 0, err
    }
    
    var c map[string]int
    json.Unmarshal(decoded, &c)
    
    return c["id"], nil
}

func GetProductsCursor(w http.ResponseWriter, r *http.Request) {
    limit := 20
    cursor := r.URL.Query().Get("cursor")
    
    var afterID int
    if cursor != "" {
        afterID, _ = decodeCursor(cursor)
    }
    
    // Get limit + 1 to check if there's more
    rows, _ := db.Query(`
        SELECT id, name FROM products
        WHERE id > ?
        ORDER BY id
        LIMIT ?
    `, afterID, limit+1)
    
    var products []Product
    for rows.Next() {
        var p Product
        rows.Scan(&p.ID, &p.Name)
        products = append(products, p)
    }
    
    hasMore := len(products) > limit
    if hasMore {
        products = products[:limit]  // Remove extra item
    }
    
    var nextCursor string
    if hasMore && len(products) > 0 {
        lastID := products[len(products)-1].ID
        nextCursor = encodeCursor(lastID)
    }
    
    response := CursorPaginatedResponse{
        Data: products,
        Pagination: CursorPagination{
            NextCursor: nextCursor,
            HasMore:    hasMore,
        },
    }
    
    json.NewEncoder(w).Encode(response)
}
```

---

## Comparison Table

| Feature | Offset | Cursor |
|---------|--------|--------|
| **Complexity** | Simple | Medium |
| **Performance** | Degrades at scale | Constant |
| **Jump to page** | ✅ Yes | ❌ No |
| **Total count** | ✅ Yes | ❌ No |
| **Consistency** | ⚠️ Can skip/duplicate | ✅ Consistent |
| **Use case** | Admin dashboards | Infinite scroll |

---

## When to Use Each

**Offset-based:**
- Admin dashboards
- Small datasets (<10k)
- Need page numbers
- Need total count

**Cursor-based:**
- Social media feeds
- Infinite scroll
- Large datasets (>100k)
- Real-time data
- High-frequency inserts

---

## Pagination Best Practices

### 1. Set Maximum Limit

```go
const MaxLimit = 100

limit := getLimit(r)
if limit > MaxLimit {
    limit = MaxLimit
}
```

Prevents: `GET /products?limit=999999`

---

### 2. Default Limit

```go
if limit < 1 {
    limit = 20  // Default
}
```

---

### 3. Include Links (HATEOAS)

```json
{
  "data": [...],
  "links": {
    "self": "/products?page=2&limit=20",
    "first": "/products?page=1&limit=20",
    "prev": "/products?page=1&limit=20",
    "next": "/products?page=3&limit=20",
    "last": "/products?page=50&limit=20"
  }
}
```

---

### 4. Consistent Ordering

```sql
❌ SELECT * FROM products LIMIT 20
✅ SELECT * FROM products ORDER BY id LIMIT 20
```

Without ORDER BY, results are unpredictable.

---

### 5. Index for Performance

```sql
CREATE INDEX idx_products_id ON products(id);
CREATE INDEX idx_created_at ON products(created_at);
```

---

## Advanced: Multi-Column Cursor

**Problem:** Sorting by non-unique column (created_at)

```sql
-- Multiple items with same created_at
-- Cursor breaks!
```

**Solution:** Include unique tiebreaker (id)

```json
{
  "cursor": {
    "created_at": "2025-01-15T10:00:00Z",
    "id": 123
  }
}
```

**SQL:**
```sql
SELECT * FROM products
WHERE (created_at, id) > ('2025-01-15T10:00:00Z', 123)
ORDER BY created_at, id
LIMIT 20
```

---

## Real-World Examples

### GitHub API
```
GET /repos/{owner}/{repo}/issues?page=2&per_page=30
```

Uses offset-based pagination.

---

### Twitter API
```
GET /tweets?max_results=10&next_token=...
```

Uses cursor-based pagination.

---

### Facebook Graph API
```
GET /me/friends?limit=25&after=MTAxNTExOTQ1MjAwNzI5NDE=
```

Uses cursor-based with `after` parameter.

---

## Practice Questions

<details>
<summary><strong>View Questions</strong></summary>

### Fill in the Blanks

1. Offset-based pagination uses __________ and __________ parameters.
2. Cursor-based pagination is better for __________ datasets.
3. The main problem with offset pagination is __________ degradation at scale.
4. Cursor values are typically __________ encoded.

### True/False

1. ⬜ Offset pagination is always faster than cursor pagination
2. ⬜ Cursor pagination can skip/duplicate items if data changes
3. ⬜ You should always allow unlimited page size
4. ⬜ ORDER BY is optional in pagination queries
5. ⬜ Cursor pagination allows jumping to any page

### Multiple Choice

1. Which SQL is most efficient for page 1000?
   - A) `LIMIT 20 OFFSET 20000`
   - B) `WHERE id > 20000 LIMIT 20`
   - C) `LIMIT 20000`
   - D) No difference

2. What's the best pagination for infinite scroll?
   - A) Offset-based
   - B) Cursor-based
   - C) No pagination
   - D) Load everything

3. What should you do if user requests `limit=9999999`?
   - A) Return all data
   - B) Return error
   - C) Cap at maximum limit
   - D) Ignore limit parameter

---

### Answers

<details>
<summary><strong>View Answers</strong></summary>

**Fill in the Blanks:**
1. page, limit (or limit, offset)
2. large
3. performance
4. base64

**True/False:**
1. ❌ False - Cursor is faster at scale
2. ❌ False - Cursor is consistent; offset can skip/duplicate
3. ❌ False - Always set maximum limit (e.g., 100)
4. ❌ False - ORDER BY is required for consistent results
5. ❌ False - Cursor only allows next/prev navigation

**Multiple Choice:**
1. **B** - WHERE with indexed column is fastest
2. **B** - Cursor-based avoids pagination issues
3. **C** - Cap at maximum (e.g., 100) to protect server

</details>

</details>