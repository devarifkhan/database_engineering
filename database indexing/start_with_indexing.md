arifulislam@ariful-islam:~$ docker exec -it pg psql -U postgres
psql (17.4 (Debian 17.4-1.pgdg120+2))
Type "help" for help.

postgres=# create table employees( id serial primary key, name text);
CREATE TABLE
postgres=# create or replace function random_string(length integer) returns text as 
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
CREATE FUNCTION
postgres=# insert into employees(name)(select random_string(10) from generate_series(0, 1000000));
INSERT 0 1000001
postgres=# \d employees;
                            Table "public.employees"
 Column |  Type   | Collation | Nullable |                Default                
--------+---------+-----------+----------+---------------------------------------
 id     | integer |           | not null | nextval('employees_id_seq'::regclass)
 name   | text    |           |          | 
Indexes:
    "employees_pkey" PRIMARY KEY, btree (id)

postgres=# select id from employees where id = 1000;
  id  
------
 1000
(1 row)

postgres=# select * from employees where id = 1000;
  id  | name 
------+------
 1000 | 8
(1 row)

postgres=# explain analyze select id from employees where id = 2000;
postgres=# explain analyze select id from employees where id = 3000;
postgres=# 
postgres=# explain analyze select name from employees where id = 5000;
postgres=# explain analyze select id from employees where name = "Zs";
ERROR:  column "Zs" does not exist
LINE 1: explain analyze select id from employees where name = "Zs";
                                                              ^
postgres=# explain analyze select id from employees where name = 'Zs';
postgres=# explain analyze select id from employees where name like '%Zs%';
postgres=# create index employees_name on employees(name);
CREATE INDEX
postgres=# explain analyze select id from employees where name = 'Zs';
postgres=# 
postgres=# explain analyze select id,name from employees where like '%Zs%';
ERROR:  type "like" does not exist
LINE 1: ...plain analyze select id,name from employees where like '%Zs%...
                                                             ^
postgres=# explain analyze select id,name from employees where name like '%Zs%';
postgres=# 
