# The SQL Query Planner and Optimizer with EXPLAIN

## Setup Instructions

```bash
# Make sure to run the container with at least 1GB shared memory
docker run --name pg --shm-size=1g -e POSTGRES_PASSWORD=postgres postgres
```

## Database Setup

### 1. Create the grades table
```sql
create table grades (
    id serial primary key, 
    g int,
    name text 
);
```

### 2. Insert initial sample data (501 rows)
```sql
insert into grades (g, name) 
select 
    random()*100,
    substring(md5(random()::text), 0, floor(random()*31)::int)
from generate_series(0, 500);
```

### 3. Update table statistics
```sql
vacuum (analyze, verbose, full);
```

### 4. Test query before index creation
```sql
explain analyze select id, g from grades where g > 80 and g < 95 order by g;
```

### 5. Create index on g column
```sql
create index grades_g on grades(g);
```

### 6. Scale up the dataset (add 2M more rows)
```sql
-- Check current count
SELECT COUNT(*) FROM grades;
-- Result: 501 rows

-- Insert 2 million additional rows
INSERT INTO grades (g, name)
SELECT
    (random() * 100)::int,
    substring(md5(random()::text), 1, 20)
FROM generate_series(1, 2000000);
-- Result: INSERT 0 2000000
```

## Query Plan Analysis

### 1. Ordered Query Using Index
```sql
explain select * from grades order by g;
```

**Query Plan:**
```
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Index Scan using grades_g on grades  (cost=0.43..92820.43 rows=1842428 width=23)
```

**Analysis:** PostgreSQL uses the `grades_g` index to retrieve data in sorted order, eliminating the need for a separate sort operation.

### 2. Ordered Query Without Suitable Index
```sql
explain select * from grades order by name;
```

**Query Plan:**
```
                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Gather Merge  (cost=125966.20..320472.91 rows=1667084 width=28)
   Workers Planned: 2
   ->  Sort  (cost=124966.17..127050.03 rows=833542 width=28)
         Sort Key: name
         ->  Parallel Seq Scan on grades  (cost=0.00..23045.42 rows=833542 width=28)
```

**Analysis:** Without an index on `name`, PostgreSQL uses parallel processing with 2 workers, performs a sequential scan, then sorts the results.

### 3. Simple Column Selection
```sql
explain select id from grades;
```

**Query Plan:**
```
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..34715.01 rows=2000501 width=4)
```

**Analysis:** Even though `id` has a primary key index, PostgreSQL chooses a sequential scan because it needs to read all rows.

### 4. Point Lookup Query
```sql
explain select * from grades where id = 10;
```

**Query Plan:**
```
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..8.45 rows=1 width=28)
   Index Cond: (id = 10)
```

**Analysis:** Perfect use case for index scan - highly selective query with exact match on indexed column.

## Query Plan Components Explained

| Component | Description | Example |
|-----------|-------------|----------|
| **Cost** | Estimated execution cost (startup..total) | `cost=0.43..8.45` |
| **Rows** | Estimated number of rows returned | `rows=1` |
| **Width** | Average row size in bytes | `width=28` |
| **Index Cond** | Condition used for index lookup | `Index Cond: (id = 10)` |
| **Sort Key** | Column(s) used for sorting | `Sort Key: name` |
| **Workers Planned** | Number of parallel workers | `Workers Planned: 2` |

## Execution Strategy Comparison

| Query Type | Strategy Used | Cost Range | Reasoning |
|------------|---------------|------------|----------|
| **ORDER BY indexed column** | Index Scan | 0.43..92820.43 | Index provides natural ordering |
| **ORDER BY non-indexed column** | Parallel Seq Scan + Sort | 125966.20..320472.91 | No index available, parallel processing helps |
| **SELECT all IDs** | Sequential Scan | 0.00..34715.01 | Reading all rows, index not beneficial |
| **WHERE id = value** | Index Scan | 0.43..8.45 | Highly selective, perfect for index |

## Optimizer Decision Factors

### When PostgreSQL Chooses Index Scan:
- **High selectivity**: Query returns small percentage of total rows
- **Indexed columns**: Suitable index exists for WHERE or ORDER BY clauses
- **Point lookups**: Exact match conditions (=, IN with few values)

### When PostgreSQL Chooses Sequential Scan:
- **Low selectivity**: Query returns large percentage of rows
- **No suitable index**: Required columns not indexed
- **Full table access**: SELECT without WHERE clause

### When PostgreSQL Uses Parallel Processing:
- **Large datasets**: Tables with significant row counts
- **CPU-intensive operations**: Sorting, aggregation
- **Available workers**: System has multiple CPU cores

## Performance Optimization Tips

1. **Create indexes on frequently queried columns**
   ```sql
   create index idx_column_name on table_name(column_name);
   ```

2. **Use EXPLAIN ANALYZE for actual performance**
   ```sql
   explain analyze select * from table_name where condition;
   ```

3. **Consider composite indexes for multi-column queries**
   ```sql
   create index idx_multi on table_name(col1, col2);
   ```

4. **Monitor query costs and adjust accordingly**
   - Low startup cost: Quick to begin execution
   - Low total cost: Efficient overall execution

5. **Update table statistics regularly**
   ```sql
   analyze table_name;
   ```

## Key Takeaways

- **PostgreSQL's optimizer is cost-based** - it chooses the plan with the lowest estimated cost
- **Index scans are not always faster** - depends on selectivity and data distribution
- **Parallel processing** can significantly improve performance for large datasets
- **Regular ANALYZE** keeps statistics current for optimal plan selection
- **EXPLAIN is essential** for understanding and optimizing query performance
