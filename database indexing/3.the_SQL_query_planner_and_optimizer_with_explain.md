--explain explained

-- make sure to run the container with at least 1gb shared memory
-- docker run --name pg —shm-size=1g -e POSTGRES_PASSWORD=postgres —name pg postgres



create table grades (
id serial primary key, 
 g int,
 name text 
); 


insert into grades (g,
name  ) 
select 
random()*100,
substring(md5(random()::text ),0,floor(random()*31)::int)
 from generate_series(0, 500);

vacuum (analyze, verbose, full);

explain analyze select id,g from grades where g > 80 and g < 95 order by g;

create index grades_g on grades(g);


postgres=#  SELECT COUNT(*) FROM grades;
 count 
-------
   501
(1 row)

postgres=# INSERT INTO grades (g, name)
SELECT
    (random() * 100)::int,
    substring(md5(random()::text), 1, 20)
FROM generate_series(1, 2000000);
INSERT 0 2000000
postgres=# explain select * from grades order by g;
                                    QUERY PLAN                                    
----------------------------------------------------------------------------------
 Index Scan using grades_g on grades  (cost=0.43..92820.43 rows=1842428 width=23)
(1 row)

postgres=# explain select * from grades order by name;
                                     QUERY PLAN                                      
-------------------------------------------------------------------------------------
 Gather Merge  (cost=125966.20..320472.91 rows=1667084 width=28)
   Workers Planned: 2
   ->  Sort  (cost=124966.17..127050.03 rows=833542 width=28)
         Sort Key: name
         ->  Parallel Seq Scan on grades  (cost=0.00..23045.42 rows=833542 width=28)
(5 rows)

postgres=# explain select id from grades;
                           QUERY PLAN                           
----------------------------------------------------------------
 Seq Scan on grades  (cost=0.00..34715.01 rows=2000501 width=4)
(1 row)

postgres=# explain select * from grades where id = 10 ;
                                QUERY PLAN                                 
---------------------------------------------------------------------------
 Index Scan using grades_pkey on grades  (cost=0.43..8.45 rows=1 width=28)
   Index Cond: (id = 10)
(2 rows)
