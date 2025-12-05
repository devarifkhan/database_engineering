# Index Scan vs Index Only Scan

## Setup and Connection

```bash
# Connect to PostgreSQL container
docker exec -it postgres_db psql -U postgres
```

## Database Setup

### 1. Create the grades table with primary key
```sql
create table grades (
    id serial primary key, 
    g int,
    name text 
);
```

### 2. Check initial table structure
```sql
\d grades;
```

**Output:**
```
                            Table "public.grades"
 Column |  Type   | Collation | Nullable |              Default               
--------+---------+-----------+----------+------------------------------------
 id     | integer |           | not null | nextval('grades_id_seq'::regclass)
 g      | integer |           |          | 
 name   | text    |           |          | 
Indexes:
    "grades_pkey" PRIMARY KEY, btree (id)
```

### 3. Remove primary key constraint for demonstration
```sql
ALTER TABLE grades DROP CONSTRAINT grades_pkey;
```

**After removal:**
```sql
\d grades;
```

**Output:**
```
                            Table "public.grades"
 Column |  Type   | Collation | Nullable |              Default               
--------+---------+-----------+----------+------------------------------------
 id     | integer |           | not null | nextval('grades_id_seq'::regclass)
 g      | integer |           |          | 
 name   | text    |           |          | 
```

### 4. Insert 500,000 sample records
```sql
insert into grades (g, name) 
select 
    random()*100,
    substring(md5(random()::text), 0, floor(random()*31)::int)
from generate_series(0, 500000);
-- Result: INSERT 0 500001
```

### 5. Test basic queries
```sql
-- Test single record lookup
select name from grades where id = 7;
-- Result: 4b7d2ff1cebd26

-- Test full record lookup
select * from grades where id = 7;
-- Result: 7 | 18 | 4b7d2ff1cebd26
```

## Index Performance Comparison

### Phase 1: No Index (Sequential Scan)

```sql
-- Query without any index on id
explain analyze select name from grades where id = 7;
```

**Expected Result:** Sequential scan through entire table to find id = 7

### Phase 2: Standard Index (Index Scan)

```sql
-- Create standard B-tree index on id
create index id_idx on grades(id);

-- Test query that needs heap lookup
explain analyze select name from grades where id = 7;

-- Test query that can be satisfied by index alone
explain analyze select id from grades where id = 7;
```

**Analysis:**
- **Query 1** (`select name`): Index Scan + Heap Lookup (needs `name` from table)
- **Query 2** (`select id`): Index Only Scan possible (id is in index)

### Phase 3: Covering Index (Index Only Scan)

```sql
-- Drop standard index
drop index id_idx;

-- Create covering index with INCLUDE clause
create index id_idx on grades(id) include (name);

-- Test query that can now be satisfied by index only
explain analyze select name from grades where id = 7;

-- Test query for column not in index
explain analyze select g from grades where id = 7;
```

**Analysis:**
- **Query 1** (`select name`): Index Only Scan (both `id` and `name` in index)
- **Query 2** (`select g`): Index Scan + Heap Lookup (`g` not in index)

## Index Scan vs Index Only Scan Comparison

| Aspect | Index Scan | Index Only Scan |
|--------|------------|------------------|
| **Definition** | Uses index to find rows, then accesses heap | Satisfies query entirely from index |
| **I/O Operations** | Index read + Heap read | Index read only |
| **Performance** | Moderate (2 I/O operations) | Fast (1 I/O operation) |
| **Requirements** | Index on WHERE clause columns | Index must contain all SELECT columns |
| **Memory Usage** | Higher (index + heap pages) | Lower (index pages only) |
| **Visibility Check** | Always checks heap for tuple visibility | May need heap check for visibility |

## Query Execution Patterns

### Standard Index Scenarios

| Query | Index Type | Execution Plan | Reason |
|-------|------------|----------------|--------|
| `SELECT name WHERE id = 7` | Standard on `id` | Index Scan + Heap Lookup | `name` not in index |
| `SELECT id WHERE id = 7` | Standard on `id` | Index Only Scan | `id` available in index |
| `SELECT * WHERE id = 7` | Standard on `id` | Index Scan + Heap Lookup | Multiple columns needed |

### Covering Index Scenarios

| Query | Index Type | Execution Plan | Reason |
|-------|------------|----------------|--------|
| `SELECT name WHERE id = 7` | Covering `id INCLUDE (name)` | Index Only Scan | Both columns in index |
| `SELECT id WHERE id = 7` | Covering `id INCLUDE (name)` | Index Only Scan | `id` in index |
| `SELECT g WHERE id = 7` | Covering `id INCLUDE (name)` | Index Scan + Heap Lookup | `g` not included |

## Performance Metrics Analysis

### Index Scan Characteristics
```sql
-- Typical execution plan
Index Scan using id_idx on grades (cost=0.42..8.44 rows=1 width=X)
  Index Cond: (id = 7)
```

**Key Metrics:**
- **Startup Cost**: Time to find first row
- **Total Cost**: Complete query execution cost
- **Rows**: Estimated rows returned
- **Width**: Average row size in bytes

### Index Only Scan Characteristics
```sql
-- Typical execution plan
Index Only Scan using id_idx on grades (cost=0.42..4.44 rows=1 width=X)
  Index Cond: (id = 7)
  Heap Fetches: 0
```

**Key Metrics:**
- **Lower Total Cost**: No heap access required
- **Heap Fetches**: Number of heap lookups (0 is optimal)
- **Faster Execution**: Single I/O operation

## When Each Scan Type Occurs

### Index Scan Triggers
- **Missing columns**: SELECT columns not in index
- **Complex expressions**: Computed columns or functions
- **Visibility checks**: When MVCC requires heap verification
- **Large result sets**: When index-only scan would be inefficient

### Index Only Scan Triggers
- **Complete coverage**: All SELECT columns in index
- **Simple conditions**: Straightforward WHERE clauses
- **Visibility map**: Table has recent VACUUM for visibility info
- **Small result sets**: Efficient for selective queries

## Optimization Strategies

### 1. Design Covering Indexes
```sql
-- For frequently queried column combinations
CREATE INDEX idx_covering ON table_name(key_col) INCLUDE (select_cols);
```

### 2. Monitor Index Usage
```sql
-- Check index scan types
SELECT 
    schemaname, tablename, indexname,
    idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes 
WHERE tablename = 'grades';
```

### 3. Analyze Query Patterns
```sql
-- Use EXPLAIN (ANALYZE, BUFFERS) for detailed analysis
EXPLAIN (ANALYZE, BUFFERS) SELECT name FROM grades WHERE id = 7;
```

## Best Practices

### Index Design Guidelines
1. **Identify frequent query patterns** before creating indexes
2. **Use covering indexes** for predictable SELECT column sets
3. **Balance storage cost** against query performance gains
4. **Regular VACUUM** to maintain visibility maps for index-only scans

### Performance Monitoring
1. **Track heap fetches** in index-only scans (should be 0 or low)
2. **Monitor buffer usage** to understand I/O patterns
3. **Compare execution times** not just cost estimates
4. **Analyze index bloat** and rebuild when necessary

## Visibility Map Importance

For index-only scans to work optimally:

```sql
-- Regular maintenance to update visibility map
VACUUM table_name;

-- Check visibility map status
SELECT 
    schemaname, tablename,
    n_tup_ins, n_tup_upd, n_tup_del,
    last_vacuum, last_autovacuum
FROM pg_stat_user_tables 
WHERE tablename = 'grades';
```

## Key Takeaways

- **Index Only Scans eliminate heap lookups** when all required columns are in the index
- **Covering indexes with INCLUDE** enable index-only scans for additional columns
- **Performance gains are significant** for selective queries with covered columns
- **Visibility maps from VACUUM** are crucial for optimal index-only scan performance
- **Storage trade-offs** must be considered when designing covering indexes
- **Query patterns should drive index design** decisions

## Advanced Considerations

- **MVCC overhead**: Index-only scans may still need heap checks for tuple visibility
- **Index maintenance**: Covering indexes require more maintenance overhead
- **Memory efficiency**: Index-only scans use less buffer pool memory
- **Concurrent access**: Better performance under high concurrency scenarios