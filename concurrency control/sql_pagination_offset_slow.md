# SQL Pagination With Offset is Very Slow

## The Problem

Using `OFFSET` for pagination becomes extremely slow with large datasets because the database must scan and skip all rows before the offset.

```sql
-- Page 1: Fast (scans 20 rows)
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 0;

-- Page 1000: Slow (scans 20,000 rows, returns 20)
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 20000;

-- Page 50000: Very Slow (scans 1,000,000 rows, returns 20)
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 1000000;
```

**Why it's slow:** Database reads and discards all offset rows before returning results.

## Setup Test Data

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Insert 1 million rows
INSERT INTO posts (title, content)
SELECT 
    'Post ' || generate_series,
    'Content for post ' || generate_series
FROM generate_series(1, 1000000);

-- Add index
CREATE INDEX idx_posts_created_at ON posts(created_at);
```

## ❌ Slow Approach: OFFSET

```sql
-- Page 50000 (offset 1,000,000)
SELECT * FROM posts 
ORDER BY id 
LIMIT 20 OFFSET 1000000;

-- Execution time: ~500ms - 2s (depending on hardware)
```

**Performance:**
```
OFFSET 0      → 1ms
OFFSET 10000  → 50ms
OFFSET 100000 → 500ms
OFFSET 1000000 → 2000ms
```

## ✅ Solution 1: Keyset Pagination (Cursor-Based)

Instead of offset, use the last seen ID as a cursor.

```sql
-- Page 1
SELECT * FROM posts 
ORDER BY id 
LIMIT 20;
-- Returns: id 1-20, last_id = 20

-- Page 2 (use last_id from previous page)
SELECT * FROM posts 
WHERE id > 20 
ORDER BY id 
LIMIT 20;
-- Returns: id 21-40, last_id = 40

-- Page 3
SELECT * FROM posts 
WHERE id > 40 
ORDER BY id 
LIMIT 20;
-- Returns: id 41-60

-- Any page (constant time!)
SELECT * FROM posts 
WHERE id > 1000000 
ORDER BY id 
LIMIT 20;
-- Execution time: ~1ms (always fast!)
```

**Performance:** O(1) - constant time regardless of page number

### With Timestamp Ordering

```sql
-- Page 1
SELECT * FROM posts 
ORDER BY created_at DESC, id DESC 
LIMIT 20;
-- Last: created_at='2024-01-15 10:30:00', id=12345

-- Page 2
SELECT * FROM posts 
WHERE (created_at, id) < ('2024-01-15 10:30:00', 12345)
ORDER BY created_at DESC, id DESC 
LIMIT 20;
```

## ✅ Solution 2: Deferred Join

Fetch only IDs with offset, then join to get full data.

```sql
-- Instead of this (slow)
SELECT * FROM posts 
ORDER BY id 
LIMIT 20 OFFSET 1000000;

-- Do this (faster)
SELECT p.* FROM posts p
INNER JOIN (
    SELECT id FROM posts 
    ORDER BY id 
    LIMIT 20 OFFSET 1000000
) AS subquery USING (id);
```

**Why faster:** Index-only scan for offset, then fetch only 20 full rows.

## ✅ Solution 3: Indexed Column Offset

Use indexed column for offset calculation.

```sql
-- Create index
CREATE INDEX idx_posts_id ON posts(id);

-- Use index for offset
SELECT * FROM posts 
WHERE id >= (
    SELECT id FROM posts 
    ORDER BY id 
    LIMIT 1 OFFSET 1000000
)
ORDER BY id 
LIMIT 20;
```

## Docker PostgreSQL Benchmark

```bash
docker exec -it postgres-deadlock psql -U postgres postgres

-- Enable timing
\timing on

-- Test OFFSET (slow)
SELECT * FROM posts ORDER BY id LIMIT 20 OFFSET 1000000;
-- Time: 1500ms

-- Test Keyset (fast)
SELECT * FROM posts WHERE id > 1000000 ORDER BY id LIMIT 20;
-- Time: 2ms
```

## Real-World Implementation

### API Response Format

```json
{
  "data": [...],
  "pagination": {
    "next_cursor": "eyJpZCI6MTIzNDV9",  // base64 encoded last_id
    "has_more": true,
    "limit": 20
  }
}
```

### Backend Code (Python Example)

```python
def get_posts(cursor=None, limit=20):
    if cursor:
        # Decode cursor (base64 encoded last_id)
        last_id = decode_cursor(cursor)
        query = "SELECT * FROM posts WHERE id > %s ORDER BY id LIMIT %s"
        params = (last_id, limit)
    else:
        query = "SELECT * FROM posts ORDER BY id LIMIT %s"
        params = (limit,)
    
    posts = execute_query(query, params)
    next_cursor = encode_cursor(posts[-1]['id']) if posts else None
    
    return {
        'data': posts,
        'next_cursor': next_cursor,
        'has_more': len(posts) == limit
    }
```

## Comparison

| Method | Page 1 | Page 1000 | Page 50000 | Jump to Page |
|--------|--------|-----------|------------|--------------|
| OFFSET | 1ms | 50ms | 2000ms | ✓ Yes |
| Keyset | 1ms | 1ms | 1ms | ✗ No |
| Deferred Join | 1ms | 30ms | 1000ms | ✓ Yes |

## When to Use Each

**Keyset Pagination (Recommended):**
- ✓ Infinite scroll
- ✓ Mobile apps
- ✓ Real-time feeds
- ✓ Large datasets
- ✗ Need page numbers

**OFFSET Pagination:**
- ✓ Small datasets (<10,000 rows)
- ✓ Need page numbers
- ✓ Jump to specific page
- ✗ Large datasets

**Deferred Join:**
- ✓ Need page numbers
- ✓ Large datasets
- ✓ Compromise solution

## Handling Backward Pagination

```sql
-- Forward (next page)
SELECT * FROM posts 
WHERE id > 1000 
ORDER BY id ASC 
LIMIT 20;

-- Backward (previous page)
SELECT * FROM (
    SELECT * FROM posts 
    WHERE id < 1000 
    ORDER BY id DESC 
    LIMIT 20
) AS subquery 
ORDER BY id ASC;
```

## Explain Analyze

```sql
-- Check query plan
EXPLAIN ANALYZE 
SELECT * FROM posts 
ORDER BY id 
LIMIT 20 OFFSET 1000000;

-- vs

EXPLAIN ANALYZE 
SELECT * FROM posts 
WHERE id > 1000000 
ORDER BY id 
LIMIT 20;
```

## Key Takeaway

**OFFSET is O(n)** - performance degrades linearly with page number.  
**Keyset is O(1)** - constant performance regardless of position.

For large datasets, always use **keyset/cursor-based pagination** instead of OFFSET. Trade-off: lose ability to jump to arbitrary pages, but gain consistent performance.
