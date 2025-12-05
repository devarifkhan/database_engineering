# Bitmap Index Scan vs Index Scan vs Table Scan

## Setup Instructions

```bash
# Make sure to run the container with at least 1GB shared memory
docker run --name pg --shm-size=1g -e POSTGRES_PASSWORD=postgres postgres
```

## Database Setup

### 1. Create the grades table with extra attributes
```sql
create table grades (
    id serial primary key,
    g int,
    name text,
    firstname text,
    lastname text,
    address text,
    bio text
);
```

### 2. Insert sample data (500 rows)
```sql
insert into grades (g, name, firstname, lastname, address, bio)
select 
    (random()*100)::int,
    substring(md5(random()::text), 0, 20),           -- name
    substring(md5(random()::text), 0, 10),           -- firstname
    substring(md5(random()::text), 0, 12),           -- lastname
    'House-' || floor(random()*100)::int 
        || ', Road-' || floor(random()*50)::int,     -- address
    'Bio: ' || substring(md5(random()::text), 0, 40) -- bio
from generate_series(1, 500);
```

### 3. Clean and analyze the table
```sql
vacuum (analyze, verbose, full);
```

### 4. Create a covering index on g including id
```sql
create index grades_g on grades(g) include(id);
```

### 5. Run explain analyze to see query plan
```sql
explain analyze
select id, g
from grades
where g > 80 and g < 95
order by g;
```

## Query Examples and Results

### Index Scan Example
```sql
postgres=# explain select name from grades where id = 1000;
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..8.45 rows=1 width=20)
   Index Cond: (id = 1000)
(2 rows)

postgres=# explain select name from grades where id < 100;
                                 QUERY PLAN                                  
-----------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..11.07 rows=94 width=20)
   Index Cond: (id < 100)
(2 rows)
```

### Sequential (Table) Scan Example
```sql
postgres=# explain select name from grades where id > 100;
                           QUERY PLAN                            
-----------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..59483.00 rows=1999905 width=20)
   Filter: (id > 100)
(2 rows)
```

### Bitmap Index Scan Example
```sql
postgres=# explain select name from grades where g > 95;
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=1501.10..36985.19 rows=80087 width=20)
   Recheck Cond: (g > 95)
   ->  Bitmap Index Scan on grades_g  (cost=0.00..1481.08 rows=80087 width=0)
         Index Cond: (g > 95)
(4 rows)
```

### Mixed Condition Example
```sql
postgres=# explain select name from grades where g > 95 and id < 10000;
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..467.23 rows=382 width=20)
   Index Cond: (id < 10000)
   Filter: (g > 95)
(3 rows)
```

## Comparison Summary

| Aspect | Index Scan | Bitmap Index Scan | Table Scan (Sequential) |
|--------|------------|-------------------|-------------------------|
| **When Used** | Small result sets, highly selective queries | Medium result sets, moderately selective | Large result sets, non-selective queries |
| **Data Access Pattern** | Random access, follows index order | Bitmap creation + sorted heap access | Sequential page-by-page reading |
| **Memory Usage** | Low | Medium (bitmap storage) | Low |
| **I/O Pattern** | Random I/O | Mixed (index + sorted heap) | Sequential I/O |
| **Best For** | Point lookups, small ranges | Range queries with moderate selectivity | Full table scans, no useful indexes |
| **Cost Characteristics** | Low startup, variable total cost | Higher startup, efficient for medium sets | High total cost, predictable |
| **Selectivity Threshold** | < 5% of table | 5-25% of table | > 25% of table |
| **Index Requirement** | Requires suitable index | Requires suitable index | No index needed |
| **Result Ordering** | Can maintain index order | No guaranteed order | No guaranteed order |
| **Parallel Execution** | Limited parallelism | Good parallelism potential | Excellent parallelism |

## Key Takeaways

- **Index Scan**: Most efficient for highly selective queries (few rows)
- **Bitmap Index Scan**: Optimal balance for moderately selective queries
- **Table Scan**: Necessary when no suitable index exists or when accessing most of the table
- PostgreSQL's query planner automatically chooses the most cost-effective method based on statistics and query characteristics
