postgres=# explain analyze select count(*) from grades_parts where g = 30;                                                             ^
postgres=# select pg_relation_size(oid) , relname from pg_class order by pg_relation_size(oid) desc;
postgres=# show ENABLE_PARTITION_PRUNING; 
 enable_partition_pruning 
--------------------------
 on
(1 row)

postgres=# set enable_partition_pruning = off;
SET
postgres=# explain select count(*) from grades_parts where g= 30;
postgres=# set enable_partition_pruning = on;
SET
postgres=# explain select count(*) from grades_parts where g= 30;
