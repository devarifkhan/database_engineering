# Combining Database Indexes for Better Performance

## Setup and Connection

```bash
# Connect to PostgreSQL container
docker exec -it postgres_db psql -U postgres
```

## Initial Table Structure

### Check existing table structure
```sql
\d test_table
```

**Output:**
```
                            Table "public.test_table"
 Column |  Type   | Collation | Nullable |                Default                 
--------+---------+-----------+----------+----------------------------------------
 id     | integer |           | not null | nextval('test_table_id_seq'::regclass)
 a      | integer |           |          | 
 b      | integer |           |          | 
 c      | integer |           |          | 
```

## Index Strategy Comparison

### Phase 1: Individual Single-Column Indexes

```sql
-- Create separate indexes on columns a and b
create index on test_table (a);
create index on test_table (b);
```

**Verify index creation:**
```sql
\d test_table
```

**Output:**
```
                            Table "public.test_table"
 Column |  Type   | Collation | Nullable |                Default                 
--------+---------+-----------+----------+----------------------------------------
 id     | integer |           | not null | nextval('test_table_id_seq'::regclass)
 a      | integer |           |          | 
 b      | integer |           |          | 
 c      | integer |           |          | 
Indexes:
    "test_table_a_idx" btree (a)
    "test_table_b_idx" btree (b)
```

### Testing Single-Column Index Performance

```sql
-- Test single column queries
explain analyze select c from test_table where a = 70;
explain analyze select c from test_table where a = 70 limit 2;
explain analyze select c from test_table where b = 100;

-- Test impossible condition (optimizer catches this)
explain analyze select c from test_table where b = 100 and b = 200;
```

**Impossible Condition Result:**
```
                                     QUERY PLAN                                     
------------------------------------------------------------------------------------
 Result  (cost=0.00..0.00 rows=0 width=0) (actual time=0.000..0.001 rows=0 loops=1)
   One-Time Filter: false
 Planning Time: 0.039 ms
 Execution Time: 0.006 ms
```

**Analysis:** PostgreSQL's optimizer recognizes that `b = 100 AND b = 200` is impossible and returns no results without executing the query.

### Testing Multi-Column Conditions with Single Indexes

```sql
-- Test AND condition (can use bitmap index scan or index intersection)
explain analyze select c from test_table where a = 100 and b = 200;

-- Test OR condition (may use bitmap heap scan)
explain analyze select c from test_table where a = 100 or b = 200;
```

### Phase 2: Composite (Multi-Column) Index

```sql
-- Drop individual indexes
drop index test_table_a_idx, test_table_b_idx;

-- Create composite index
create index on test_table (a, b);
```

**Verify composite index:**
```sql
\d test_table
```

**Output:**
```
                            Table "public.test_table"
 Column |  Type   | Collation | Nullable |                Default                 
--------+---------+-----------+----------+----------------------------------------
 id     | integer |           | not null | nextval('test_table_id_seq'::regclass)
 a      | integer |           |          | 
 b      | integer |           |          | 
 c      | integer |           |          | 
Indexes:
    "test_table_a_b_idx" btree (a, b)
```

### Testing Composite Index Performance

```sql
-- Test leading column (a) - should use index efficiently
explain analyze select c from test_table where a = 70;

-- Test non-leading column (b) - less efficient
explain analyze select c from test_table where b = 100;

-- Test both columns - optimal for composite index
explain analyze select c from test_table where a = 100 and b = 80;
```

### Phase 3: Hybrid Approach (Composite + Individual)

```sql
-- Add individual index on b for better single-column performance
create index on test_table (b);
```

**Final index structure:**
```sql
\d test_table
```

**Output:**
```
                            Table "public.test_table"
 Column |  Type   | Collation | Nullable |                Default                 
--------+---------+-----------+----------+----------------------------------------
 id     | integer |           | not null | nextval('test_table_id_seq'::regclass)
 a      | integer |           |          | 
 b      | integer |           |          | 
 c      | integer |           |          | 
Indexes:
    "test_table_a_b_idx" btree (a, b)
    "test_table_b_idx" btree (b)
```

## Index Strategy Comparison

| Strategy | Indexes | Storage Cost | Query Performance |
|----------|---------|--------------|-------------------|
| **Individual** | `(a)`, `(b)` | Medium | Good for single columns, moderate for AND/OR |
| **Composite** | `(a,b)` | Low | Excellent for `a` and `a,b`, poor for `b` only |
| **Hybrid** | `(a,b)`, `(b)` | High | Excellent for all query patterns |

## Query Performance Analysis

### Single Column Queries

| Query Pattern | Individual Indexes | Composite Index | Hybrid Approach |
|---------------|-------------------|-----------------|------------------|
| `WHERE a = X` | Uses `a_idx` | Uses `(a,b)_idx` efficiently | Uses `(a,b)_idx` efficiently |
| `WHERE b = X` | Uses `b_idx` | Sequential scan or inefficient index scan | Uses `b_idx` efficiently |

### Multi-Column Queries

| Query Pattern | Individual Indexes | Composite Index | Hybrid Approach |
|---------------|-------------------|-----------------|------------------|
| `WHERE a = X AND b = Y` | Bitmap Index Scan or Index Intersection | Optimal index scan | Optimal index scan |
| `WHERE a = X OR b = Y` | Bitmap Heap Scan | Sequential scan likely | Bitmap Heap Scan |
| `ORDER BY a, b` | External sort needed | Natural index order | Natural index order |

## Index Combination Techniques

### 1. Bitmap Index Scan
```sql
-- PostgreSQL can combine multiple single-column indexes
-- Creates bitmaps from each index and combines them
EXPLAIN ANALYZE SELECT * FROM test_table WHERE a = 100 AND b = 200;
```

**Typical Plan:**
```
Bitmap Heap Scan on test_table
  Recheck Cond: ((a = 100) AND (b = 200))
  ->  BitmapAnd
        ->  Bitmap Index Scan on test_table_a_idx
              Index Cond: (a = 100)
        ->  Bitmap Index Scan on test_table_b_idx
              Index Cond: (b = 200)
```

### 2. Index Intersection
```sql
-- PostgreSQL may use index intersection for AND conditions
-- More efficient than bitmap scans for highly selective queries
```

### 3. Composite Index Usage
```sql
-- Composite indexes follow left-to-right column order
-- Index (a,b) can efficiently handle:
-- WHERE a = X
-- WHERE a = X AND b = Y
-- ORDER BY a, b
```

## Best Practices for Index Combination

### 1. Analyze Query Patterns
```sql
-- Identify most frequent query patterns
SELECT query, calls, mean_exec_time 
FROM pg_stat_statements 
WHERE query LIKE '%test_table%'
ORDER BY calls DESC;
```

### 2. Column Order in Composite Indexes
```sql
-- Order by selectivity (most selective first)
-- Consider query frequency
CREATE INDEX idx_optimal ON test_table (high_selectivity_col, low_selectivity_col);
```

### 3. Monitor Index Usage
```sql
-- Check which indexes are actually used
SELECT 
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes 
WHERE relname = 'test_table';
```

## Index Strategy Decision Matrix

### When to Use Individual Indexes
- **Diverse query patterns** with different column combinations
- **Frequent single-column queries** on multiple columns
- **OR conditions** are common
- **Storage is not a primary concern**

### When to Use Composite Indexes
- **Predictable query patterns** with consistent column combinations
- **Frequent multi-column AND conditions**
- **Storage efficiency** is important
- **Sorting** on multiple columns is common

### When to Use Hybrid Approach
- **Mixed query workload** with both single and multi-column queries
- **Performance is critical** and storage cost is acceptable
- **Complex application** with unpredictable query patterns

## Performance Optimization Tips

### 1. Index Maintenance
```sql
-- Regular statistics updates
ANALYZE test_table;

-- Monitor index bloat
SELECT 
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
WHERE relname = 'test_table';
```

### 2. Query Optimization
```sql
-- Use EXPLAIN (ANALYZE, BUFFERS) for detailed analysis
EXPLAIN (ANALYZE, BUFFERS) 
SELECT c FROM test_table WHERE a = 100 AND b = 200;
```

### 3. Index Design Guidelines
- **Start with most frequent queries**
- **Consider column cardinality** (uniqueness)
- **Test with realistic data volumes**
- **Monitor and adjust** based on actual usage

## Common Pitfalls to Avoid

1. **Over-indexing**: Too many indexes slow down writes
2. **Wrong column order**: Inefficient composite indexes
3. **Unused indexes**: Waste storage and maintenance overhead
4. **Ignoring query patterns**: Indexes that don't match actual usage

## Key Takeaways

- **PostgreSQL can combine multiple indexes** using bitmap scans
- **Composite indexes are most efficient** for predictable multi-column queries
- **Column order matters** in composite indexes (left-to-right usage)
- **Hybrid approaches** offer flexibility at the cost of storage
- **Regular monitoring** is essential for optimal index strategy
- **Query patterns should drive** index design decisions

## Advanced Considerations

- **Partial indexes**: For queries with consistent WHERE conditions
- **Expression indexes**: For computed columns or functions
- **Covering indexes**: Include frequently selected columns
- **Index-only scans**: Minimize heap access for better performance
