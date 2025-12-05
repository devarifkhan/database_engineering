# How Database Optimizers Decide to Use Indexes

## Overview

Database optimizers are cost-based systems that analyze multiple execution plans and choose the one with the lowest estimated cost. Understanding how they make index usage decisions is crucial for database performance optimization.

## The Optimizer Decision Process

### 1. Query Analysis Phase
```sql
-- Example query for analysis
SELECT name, email FROM users WHERE age > 25 AND city = 'New York' ORDER BY created_at;
```

**Optimizer Steps:**
1. **Parse the query** and identify tables, columns, and conditions
2. **Identify available indexes** on referenced columns
3. **Generate multiple execution plans** using different index combinations
4. **Estimate costs** for each plan
5. **Select the lowest-cost plan**

### 2. Cost Estimation Factors

| Factor | Description | Impact on Decision |
|--------|-------------|-------------------|
| **Selectivity** | Percentage of rows matching condition | High selectivity favors index usage |
| **Cardinality** | Number of unique values in column | Higher cardinality improves index effectiveness |
| **Table Size** | Total number of rows | Larger tables benefit more from indexes |
| **Index Size** | Size of index vs table | Smaller indexes are preferred |
| **I/O Cost** | Disk access patterns | Sequential vs random I/O costs |
| **CPU Cost** | Processing overhead | Index traversal vs full scan processing |

## Selectivity Analysis

### High Selectivity (Index Favored)
```sql
-- Returns ~0.1% of rows - optimizer likely chooses index
SELECT * FROM orders WHERE order_id = 12345;

-- Returns ~1% of rows - index still beneficial
SELECT * FROM users WHERE email = 'john@example.com';
```

### Low Selectivity (Sequential Scan Favored)
```sql
-- Returns ~80% of rows - sequential scan likely chosen
SELECT * FROM users WHERE status = 'active';

-- Returns ~50% of rows - depends on other factors
SELECT * FROM orders WHERE created_at > '2023-01-01';
```

## Index Usage Decision Matrix

| Condition | Selectivity | Table Size | Optimizer Choice | Reasoning |
|-----------|-------------|------------|------------------|-----------|
| `id = 123` | Very High (<0.1%) | Any | Index Scan | Point lookup, highly selective |
| `status = 'active'` | Low (>50%) | Small (<1000 rows) | Sequential Scan | Small table, index overhead not worth it |
| `status = 'active'` | Low (>50%) | Large (>1M rows) | Sequential Scan | Too many rows, index not selective |
| `age BETWEEN 25 AND 35` | Medium (5-20%) | Large | Index Scan | Good selectivity on large table |
| `name LIKE 'John%'` | Variable | Any | Index Scan | Prefix match can use index |
| `name LIKE '%John%'` | Variable | Any | Sequential Scan | Wildcard prefix prevents index usage |

## Cost Calculation Examples

### Example 1: Point Lookup
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM users WHERE id = 1000;
```

**Cost Factors:**
- **Index traversal**: 3-4 page reads (B-tree depth)
- **Heap lookup**: 1 page read
- **Total I/O**: ~4-5 pages
- **Estimated cost**: 8.45

### Example 2: Range Query
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE created_at > '2024-01-01';
```

**Cost Factors:**
- **Index scan**: Multiple pages (depends on range)
- **Heap lookups**: One per matching row
- **Total cost**: Depends on selectivity
- **Break-even point**: Usually around 5-15% of table

### Example 3: Full Table Scan
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT COUNT(*) FROM users;
```

**Cost Factors:**
- **Sequential I/O**: All table pages
- **No random access**: More efficient disk usage
- **Parallel processing**: Can utilize multiple workers
- **Lower per-page cost**: Sequential reads are faster

## Statistics and Their Impact

### Table Statistics
```sql
-- View table statistics
SELECT 
    schemaname, tablename, n_live_tup, n_dead_tup,
    last_vacuum, last_autovacuum, last_analyze, last_autoanalyze
FROM pg_stat_user_tables 
WHERE tablename = 'users';
```

### Column Statistics
```sql
-- View column statistics
SELECT 
    attname, n_distinct, correlation,
    most_common_vals, most_common_freqs
FROM pg_stats 
WHERE tablename = 'users' AND attname = 'status';
```

**Key Statistics:**
- **n_distinct**: Number of unique values (affects cardinality estimates)
- **correlation**: Physical vs logical ordering (affects I/O patterns)
- **most_common_vals**: Frequent values (affects selectivity calculations)
- **histogram_bounds**: Value distribution (for range queries)

## Optimizer Configuration Parameters

### Cost Parameters
```sql
-- View current cost settings
SHOW seq_page_cost;      -- Default: 1.0
SHOW random_page_cost;   -- Default: 4.0
SHOW cpu_tuple_cost;     -- Default: 0.01
SHOW cpu_index_tuple_cost; -- Default: 0.005
SHOW cpu_operator_cost;  -- Default: 0.0025
```

### Tuning for SSD Storage
```sql
-- Reduce random page cost for SSD
SET random_page_cost = 1.1;  -- Closer to sequential cost

-- This makes indexes more attractive to the optimizer
```

## Index Usage Patterns

### When Optimizer Chooses Index Scan
- **High selectivity**: Query returns <5% of rows
- **Point lookups**: Equality conditions on indexed columns
- **Range queries**: With good selectivity on indexed columns
- **Ordered results**: When index provides natural ordering

### When Optimizer Chooses Sequential Scan
- **Low selectivity**: Query returns >25% of rows
- **Small tables**: Index overhead exceeds benefit
- **Missing statistics**: Outdated or missing table statistics
- **Complex expressions**: Functions or calculations on indexed columns

### When Optimizer Uses Bitmap Scans
- **Medium selectivity**: Query returns 5-25% of rows
- **Multiple conditions**: Can combine multiple indexes
- **Large result sets**: More efficient than many random I/Os

## Forcing Index Usage (When Necessary)

### Disable Sequential Scans (Testing Only)
```sql
-- Force index usage for testing
SET enable_seqscan = off;
EXPLAIN ANALYZE SELECT * FROM users WHERE status = 'active';
SET enable_seqscan = on;  -- Always reset
```

### Index Hints (PostgreSQL doesn't support, but concept)
```sql
-- Other databases support hints like:
-- SELECT /*+ INDEX(users, idx_status) */ * FROM users WHERE status = 'active';
```

## Monitoring Index Usage

### Index Usage Statistics
```sql
SELECT 
    schemaname, tablename, indexname,
    idx_scan, idx_tup_read, idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
ORDER BY idx_scan DESC;
```

### Unused Indexes
```sql
SELECT 
    schemaname, tablename, indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes 
WHERE idx_scan = 0
AND indexrelname NOT LIKE '%_pkey';
```

## Common Optimizer Mistakes and Solutions

### Problem 1: Outdated Statistics
```sql
-- Symptom: Poor execution plans despite good indexes
-- Solution: Update statistics
ANALYZE table_name;

-- Or enable auto-analyze
ALTER TABLE table_name SET (autovacuum_analyze_scale_factor = 0.1);
```

### Problem 2: Function-Based Conditions
```sql
-- Problem: Index not used
SELECT * FROM users WHERE UPPER(name) = 'JOHN';

-- Solution: Create expression index
CREATE INDEX idx_users_upper_name ON users (UPPER(name));
```

### Problem 3: Implicit Type Conversions
```sql
-- Problem: Index not used due to type mismatch
SELECT * FROM users WHERE user_id = '123';  -- user_id is integer

-- Solution: Use correct type
SELECT * FROM users WHERE user_id = 123;
```

## Advanced Optimizer Features

### Parallel Query Processing
```sql
-- Optimizer may choose parallel sequential scan over index
-- for large result sets
SET max_parallel_workers_per_gather = 4;
EXPLAIN (ANALYZE, BUFFERS) SELECT COUNT(*) FROM large_table;
```

### Partial Index Consideration
```sql
-- Optimizer considers partial indexes when conditions match
CREATE INDEX idx_active_users ON users (created_at) 
WHERE status = 'active';

-- This query can use the partial index
SELECT * FROM users 
WHERE status = 'active' AND created_at > '2024-01-01';
```

### Multi-Column Index Usage
```sql
-- Index on (a, b, c) can be used for:
-- WHERE a = 1                    ✓ (leftmost column)
-- WHERE a = 1 AND b = 2          ✓ (left-to-right)
-- WHERE a = 1 AND b = 2 AND c = 3 ✓ (all columns)
-- WHERE b = 2                    ✗ (skips leftmost)
-- WHERE a = 1 AND c = 3          ✓ (partial usage)
```

## Best Practices for Optimizer Efficiency

### 1. Maintain Current Statistics
```sql
-- Schedule regular ANALYZE
SELECT cron.schedule('analyze-tables', '0 2 * * *', 'ANALYZE;');
```

### 2. Monitor Query Performance
```sql
-- Enable query statistics
ALTER SYSTEM SET shared_preload_libraries = 'pg_stat_statements';
ALTER SYSTEM SET pg_stat_statements.track = 'all';
```

### 3. Design Indexes Based on Query Patterns
```sql
-- Analyze actual query workload
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    rows
FROM pg_stat_statements 
ORDER BY total_exec_time DESC
LIMIT 10;
```

### 4. Test Index Effectiveness
```sql
-- Before creating index
EXPLAIN (ANALYZE, BUFFERS) your_query;

-- After creating index
CREATE INDEX your_index ON table_name (columns);
EXPLAIN (ANALYZE, BUFFERS) your_query;

-- Compare execution times and I/O
```

## Key Takeaways

- **Cost-based optimization**: Optimizer chooses lowest estimated cost plan
- **Selectivity is crucial**: High selectivity favors index usage
- **Statistics matter**: Outdated statistics lead to poor decisions
- **Table size impacts**: Larger tables benefit more from indexes
- **I/O patterns**: Sequential scans better for large result sets
- **Regular maintenance**: ANALYZE and VACUUM keep optimizer informed
- **Monitor and adjust**: Use pg_stat_statements to track performance
- **Test thoroughly**: Always verify optimizer decisions with EXPLAIN ANALYZE

## Troubleshooting Optimizer Decisions

### When Index Isn't Used Despite Expectations
1. **Check statistics**: `ANALYZE table_name;`
2. **Verify selectivity**: Use `EXPLAIN` to see row estimates
3. **Check index definition**: Ensure it matches query pattern
4. **Review cost parameters**: Adjust for your hardware
5. **Consider data distribution**: Skewed data affects decisions
6. **Test with different conditions**: Verify index effectiveness

The database optimizer is sophisticated but not perfect. Understanding its decision-making process helps you design better indexes and write more efficient queries.