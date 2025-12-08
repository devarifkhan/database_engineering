postgres=# \d g80100;
               Table "public.g80100"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 g      | integer |           | not null | 
Partition of: grades_parts FOR VALUES FROM (80) TO (100)

postgres=# insert into grades_parts select * from grades_org;
INSERT 0 1000001
postgres=# select count(*) from grades_parts;
  count  
---------
 1000001
(1 row)

postgres=# select max(g) from grades_parts;
 max 
-----
  99
(1 row)

postgres=# select max(g) from g0035;
 max 
-----
  34
(1 row)

postgres=# select count(*) from g0035;
 count  
--------
 350147
(1 row)

postgres=# select count(*) from g3560;
 count  
--------
 250413
(1 row)

postgres=# select max(g) from g3560;
 max 
-----
  59
(1 row)

postgres=# create index grades_parts_idx on grades_parts(g);
CREATE INDEX
postgres=# \d grades_parts;
                      Partitioned table "public.grades_parts"
 Column |  Type   | Collation | Nullable |                 Default                  
--------+---------+-----------+----------+------------------------------------------
 id     | integer |           | not null | nextval('grades_parts_id_seq'::regclass)
 g      | integer |           | not null | 
Partition key: RANGE (g)
Indexes:
    "grades_parts_idx" btree (g)
Number of partitions: 4 (Use \d+ to list them.)

