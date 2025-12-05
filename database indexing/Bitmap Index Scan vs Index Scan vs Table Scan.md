-- Make sure to run the container with at least 1GB shared memory
-- docker run --name pg --shm-size=1g -e POSTGRES_PASSWORD=postgres postgres

-- 1️⃣ Create the grades table with extra attributes
create table grades (
    id serial primary key,
    g int,
    name text,
    firstname text,
    lastname text,
    address text,
    bio text
);

-- 2️⃣ Insert sample data (500 rows)
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

-- 3️⃣ Clean and analyze the table
vacuum (analyze, verbose, full);

-- 4️⃣ Create a covering index on g including id
create index grades_g on grades(g) include(id);

-- 5️⃣ Run explain analyze to see query plan
explain analyze
select id, g
from grades
where g > 80 and g < 95
order by g;


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

postgres=# explain select name from grades where id > 100;
                           QUERY PLAN                            
-----------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..59483.00 rows=1999905 width=20)
   Filter: (id > 100)
(2 rows)

postgres=# explain select name from grades where g > 95;
                                  QUERY PLAN                                  
------------------------------------------------------------------------------
 Bitmap Heap Scan on grades  (cost=1501.10..36985.19 rows=80087 width=20)
   Recheck Cond: (g > 95)
   ->  Bitmap Index Scan on grades_g  (cost=0.00..1481.08 rows=80087 width=0)
         Index Cond: (g > 95)
(4 rows)

postgres=# explain select name from grades where g > 95 and id < 10000;
                                  QUERY PLAN                                   
-------------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..467.23 rows=382 width=20)
   Index Cond: (id < 10000)
   Filter: (g > 95)
(3 rows)

postgres=# 
