arifulislam@ariful-islam:~$ docker run --name pgmain -d -e POSTGRES_PASSWORD=postgres postgres
02ff86260a184f8bd990f4cdefb686f4a696810f7701aeb801d38d4e12aa1d5c
arifulislam@ariful-islam:~$ docker exec -it pgmain bash
root@02ff86260a18:/# psql -U postgres
psql (17.4 (Debian 17.4-1.pgdg120+2))
Type "help" for help.

postgres=# create table grades_org (id serial not null , g int not null );
CREATE TABLE
postgres=# insert into grades_org(g) select floor(random()*100) from generate_series (0,1000000);
INSERT 0 1000001
postgres=# create index grades_org_index on grades_org(g);
CREATE INDEX
