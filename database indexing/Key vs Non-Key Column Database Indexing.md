# Key vs Non-Key Column Database Indexing

## Setup and Connection

```bash
# Connect to PostgreSQL container
docker exec -it postgres_db psql -U postgres
```

## Database Setup

### 1. Create a comprehensive students table
```sql
create table students (
    id serial primary key, 
    g int,
    firstname text, 
    lastname text, 
    middlename text,
    address text,
    bio text,
    dob date,
    id1 int,
    id2 int,
    id3 int,
    id4 int,
    id5 int,
    id6 int,
    id7 int,
    id8 int,
    id9 int
);
```

### 2. Insert 50 million sample records
```sql
insert into students (
    g, firstname, lastname, middlename, address, bio, dob,
    id1, id2, id3, id4, id5, id6, id7, id8, id9
) 
select 
    random()*100,
    substring(md5(random()::text), 0, floor(random()*31)::int),
    substring(md5(random()::text), 0, floor(random()*31)::int),
    substring(md5(random()::text), 0, floor(random()*31)::int),
    substring(md5(random()::text), 0, floor(random()*31)::int),
    substring(md5(random()::text), 0, floor(random()*31)::int),
    now(),
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000,
    random()*100000
from generate_series(0, 50000000);
-- Result: INSERT 0 50000001
```

### 3. Verify data insertion
```sql
select count(*) from students;
-- Result: 50000001 rows
```

## Index Performance Testing

### Phase 1: Testing Without Index

```sql
-- Test range query without index
explain analyze select id, g from students where g > 8 and g < 95;

-- Test range query with ordering without index
explain analyze select id, g from students where g > 8 and g < 95 order by g desc;
```

### Phase 2: Create Standard Index (Key Column Only)

```sql
-- Create standard B-tree index on g column
create index g_idx on students(g);
```

**Testing with Standard Index:**
```sql
-- Test range query with standard index
explain analyze select id, g from students where g > 8 and g < 95 order by g desc;

-- Test with LIMIT clause
explain analyze select id, g from students where g > 8 and g < 95 order by g desc limit 1000;

-- Test with buffer analysis
explain (analyze,buffers) select id, g from students where g > 8 and g < 95 order by g desc limit 1000;

-- Test more selective range
explain (analyze,buffers) select id, g from students where g > 10 and g < 20 order by g desc limit 1000;

-- Test without LIMIT
explain (analyze,buffers) select id, g from students where g > 10 and g < 20 order by g desc;
```

### Phase 3: Create Covering Index (Key + Non-Key Columns)

```sql
-- Drop the standard index
drop index g_idx;

-- Create covering index with INCLUDE clause
create index g_idx on students(g) include (id);
```

**Testing with Covering Index:**
```sql
-- Test the same query with covering index
explain (analyze,buffers) select id, g from students where g > 10 and g < 20 order by g desc;
```

### Phase 4: Database Maintenance

```sql
-- Perform vacuum to update statistics and clean up
vacuum (verbose) students;
```

**Vacuum Output Analysis:**
```
INFO:  vacuuming "postgres.public.students"
INFO:  launched 1 parallel vacuum worker for index cleanup (planned: 1)
INFO:  finished vacuuming "postgres.public.students": index scans: 0
pages: 0 removed, 953337 remain, 1 scanned (0.00% of total)
tuples: 0 removed, 50000000 remain, 0 are dead but not yet removable
avg read rate: 34.781 MB/s, avg write rate: 0.000 MB/s
buffer usage: 67 hits, 238 misses, 0 dirtied
```

## Key vs Non-Key Column Concepts

### Standard Index (Key Columns Only)
```sql
create index g_idx on students(g);
```

| Aspect | Description |
|--------|-------------|
| **Structure** | B-tree with `g` values as keys |
| **Storage** | Index stores only `g` values + row pointers |
| **Query Coverage** | Can satisfy WHERE and ORDER BY on `g` |
| **Additional Lookups** | Requires heap lookup for other columns (like `id`) |
| **Size** | Smaller index size |

### Covering Index (Key + Non-Key Columns)
```sql
create index g_idx on students(g) include (id);
```

| Aspect | Description |
|--------|-------------|
| **Structure** | B-tree with `g` as key, `id` as included data |
| **Storage** | Index stores `g` values + `id` values + row pointers |
| **Query Coverage** | Can satisfy WHERE/ORDER BY on `g` AND SELECT `id` |
| **Additional Lookups** | No heap lookup needed for included columns |
| **Size** | Larger index size due to included columns |

## Performance Comparison

| Query Type | Standard Index | Covering Index | Performance Gain |
|------------|----------------|----------------|------------------|
| **WHERE g = value** | Index Scan + Heap Lookup | Index-Only Scan | Significant |
| **WHERE g BETWEEN x AND y** | Index Scan + Heap Lookup | Index-Only Scan | Moderate to High |
| **SELECT id, g WHERE g > x ORDER BY g** | Index Scan + Heap Lookup | Index-Only Scan | High |
| **SELECT * WHERE g = value** | Index Scan + Heap Lookup | Index Scan + Heap Lookup | No difference |

## When to Use Each Approach

### Use Standard Index When:
- **Query selectivity is high** (returns few rows)
- **Index size is a concern** (storage constraints)
- **Queries need many columns** not practical to include all
- **Write performance is critical** (smaller indexes = faster writes)

### Use Covering Index When:
- **Queries consistently access same column set**
- **Read performance is more critical than write performance**
- **Heap lookups are expensive** (large rows, poor cache locality)
- **Query workload is predictable** and benefits from coverage

## Buffer Analysis Insights

The `explain (analyze,buffers)` command provides crucial performance metrics:

| Metric | Description | Optimization Impact |
|--------|-------------|--------------------|
| **Buffer Hits** | Pages found in memory | Higher = better performance |
| **Buffer Misses** | Pages read from disk | Lower = better performance |
| **Pages Dirtied** | Pages modified in memory | Indicates write activity |

## Best Practices

### 1. Index Design Strategy
```sql
-- For frequently queried column combinations
create index idx_covering on table_name(key_col) include (frequently_selected_cols);

-- For highly selective queries
create index idx_standard on table_name(selective_column);
```

### 2. Performance Testing
```sql
-- Always test with realistic data volumes
explain (analyze, buffers) your_query;

-- Compare different index strategies
-- Measure actual execution time, not just cost estimates
```

### 3. Maintenance Considerations
```sql
-- Regular statistics updates
analyze table_name;

-- Periodic cleanup
vacuum table_name;

-- Monitor index usage
select * from pg_stat_user_indexes where relname = 'your_table';
```

## Key Takeaways

- **Covering indexes eliminate heap lookups** for included columns
- **Standard indexes are smaller** but may require additional I/O
- **Query patterns should drive index design** decisions
- **Buffer analysis reveals true I/O costs** beyond query planner estimates
- **Regular maintenance** (VACUUM, ANALYZE) is essential for optimal performance
- **Test with production-like data volumes** for accurate performance assessment

## Advanced Considerations

- **Index bloat**: Covering indexes can become bloated faster
- **Write performance**: More columns = slower INSERT/UPDATE operations
- **Storage costs**: Balance query performance against storage requirements
- **Maintenance overhead**: Larger indexes require more maintenance resources