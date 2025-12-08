arifulislam@ariful-islam:~$  docker exec -it pgmain bash
root@02ff86260a18:/# psql -U postgres
psql (17.4 (Debian 17.4-1.pgdg120+2))
Type "help" for help.

postgres=# create table grades_parts (id serial not null, g int not null ) partition by range(g);
CREATE TABLE
postgres=# create table g0035 (like grades_parts including indexes);
CREATE TABLE
postgres=# \d g0035;
               Table "public.g0035"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 g      | integer |           | not null | 

postgres=# create table g3560 (like grades_parts including indexes);
CREATE TABLE
postgres=# create table g6080 (like grades_parts including indexes);
CREATE TABLE
postgres=# create table g80100 (like grades_parts including indexes);
CREATE TABLE                                        ^
postgres=# alter table grades_parts attach partition g0035 for values from (0) to (35);
ALTER TABLE
postgres=# alter table grades_parts attach partition g3560 for values from (35) to (60);
ALTER TABLE
postgres=# alter table grades_parts attach partition g6080 for values from (60) to (80);
ALTER TABLE
postgres=# alter table grades_parts attach partition g80100 for values from (80) to (100);
ALTER TABLE
postgres=# \d g80100;
               Table "public.g80100"
 Column |  Type   | Collation | Nullable | Default 
--------+---------+-----------+----------+---------
 id     | integer |           | not null | 
 g      | integer |           | not null | 
Partition of: grades_parts FOR VALUES FROM (80) TO (100)

postgres=# 