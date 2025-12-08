# Getting Started with Database Indexing

## Setup and Connection

```bash
# Connect to PostgreSQL container
docker exec -it pg psql -U postgres
```

## Database Setup

### 1. Create the employees table
```sql
create table employees( 
    id serial primary key, 
    name text
);
```

### 2. Create a random string generator function
```sql
create or replace function random_string(length integer) returns text as 
$$
declare
  chars text[] := '{0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z}';
  result text := '';
  i integer := 0;
  length2 integer := (select trunc(random() * length + 1));
begin
  if length2 < 0 then
    raise exception 'Given length cannot be less than 0';
  end if;
  for i in 1..length2 loop
    result := result || chars[1+random()*(array_length(chars, 1)-1)];
  end loop;
  return result;
end;
$$ language plpgsql;
```

### 3. Insert 1 million sample records
```sql
insert into employees(name)
(select random_string(10) from generate_series(0, 1000000));
-- Result: INSERT 0 1000001
```

### 4. Check table structure
```sql
\d employees;
```

**Output:**
```
                            Table "public.employees"
 Column |  Type   | Collation | Nullable |                Default                
--------+---------+-----------+----------+---------------------------------------
 id     | integer |           | not null | nextval('employees_id_seq'::regclass)
 name   | text    |           |          | 
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)
```

## Query Performance Testing

### Basic Queries
```sql
-- Simple ID lookup (uses primary key index)
select id from employees where id = 1000;
select * from employees where id = 1000;
```

### Performance Analysis Before Index
```sql
-- These queries will show performance without name index
explain analyze select id from employees where id = 2000;
explain analyze select name from employees where id = 5000;
explain analyze select id from employees where name = 'Zs';
explain analyze select id from employees where name like '%Zs%';
```

### Creating Index on Name Column
```sql
create index employees_name on employees(name);
```

### Performance Analysis After Index
```sql
-- Test performance with name index
explain analyze select id from employees where name = 'Zs';
explain analyze select id,name from employees where name like '%Zs%';
```


## Key Concepts Summary

| Concept | Description | Example |
|---------|-------------|----------|
| **Primary Key Index** | Automatically created unique index | `id serial primary key` creates btree index |
| **Secondary Index** | Manual index on non-primary columns | `create index employees_name on employees(name)` |
| **Index Scan** | Fast lookup using index structure | Queries with `WHERE id = value` |
| **Sequential Scan** | Full table scan without index | Queries on unindexed columns |
| **LIKE Patterns** | Pattern matching with wildcards | `name like '%Zs%'` (may not use index efficiently) |
| **Exact Match** | Precise value lookup | `name = 'Zs'` (can use index efficiently) |

## Performance Impact Analysis

| Query Type | Before Name Index | After Name Index | Performance Gain |
|------------|-------------------|------------------|------------------|
| **ID Lookup** | Fast (uses PK index) | Fast (uses PK index) | No change |
| **Name Exact Match** | Slow (sequential scan) | Fast (index scan) | Significant improvement |
| **Name Pattern Match** | Slow (sequential scan) | Slow (index not effective for `%pattern%`) | Minimal improvement |
| **ID Range** | Fast (uses PK index) | Fast (uses PK index) | No change |

## Best Practices Learned

1. **Always use single quotes** for string literals in SQL
2. **Primary keys automatically get indexes** - no manual creation needed
3. **Create indexes on frequently queried columns** to improve performance
4. **Pattern matching with leading wildcards** (`%pattern%`) cannot effectively use indexes
5. **Use EXPLAIN ANALYZE** to understand query execution plans
6. **Test performance before and after** creating indexes to measure impact

## Next Steps

- Experiment with different index types (btree, hash, gin, gist)
- Test composite indexes on multiple columns
- Analyze the trade-offs between query speed and insert/update performance
- Learn about partial indexes and expression indexes
