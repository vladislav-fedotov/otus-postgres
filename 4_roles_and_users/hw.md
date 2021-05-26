1. создайте новый кластер PostgresSQL 13 (на выбор - GCE, CloudSQL)
```
gcloud compute instances create postgres-roles \
        --zone=$ZONE \
        --image=$IMAGE \
        --image-project=ubuntu-os-cloud \
        --maintenance-policy=MIGRATE \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=10GB \
        --boot-disk-type=pd-ssd
```

2. зайдите в созданный кластер под пользователем postgres
```
sudo apt update && \
sudo apt upgrade -y && \
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && \
wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && \
sudo apt-get update && \
sudo apt-get -y install postgresql && sudo apt install unzip
```
`sudo -u postgres psql`

3. создайте новую базу данных testdb 

`CREATE DATABASE testdb;`

4. зайдите в созданную базу данных под пользователем postgres

`\c testdb postgres`

5. создайте новую схему testnm 

`CREATE SCHEMA testnm;`

6. создайте новую таблицу t1 с одной колонкой c1 типа integer 

`CREATE TABLE t1 (c1 INTEGER);`

7. вставьте строку со значением c1=1 

`INSERT INTO t1 VALUES (1);`

8. создайте новую роль readonly

`CREATE ROLE readonly;`

9. дайте новой роли право на подключение к базе данных testdb 

`GRANT CONNECT ON DATABASE testdb TO readonly;`

10. дайте новой роли право на использование схемы testnm

`GRANT USAGE ON SCHEMA testnm TO readonly;`

11. дайте новой роли право на select для всех таблиц схемы testnm 

`GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;`

12. создайте пользователя testread с паролем test123

`CREATE USER testread WITH PASSWORD 'test123';`

13. дайте роль readonly пользователю testread 

`GRANT readonly TO testread;`

14. зайдите под пользователем testread в базу данных testdb
```
show hba_file;
sudo nano /etc/postgresql/13/main/pg_hba.conf  (поменять peer authentification на md5)
sudo su postgres
psql -h 127.0.0.1 -U testread -d testdb -W
```
15. сделайте select * from t1;

`ERROR:  permission denied for table t1`

16. получилось? (могло если вы делали сами не по шпаргалке и не упустили один существенный момент про который позже)

Нет

17. напишите что именно произошло в тексте домашнего задания

У пользователя `testdb` не хватило прав на чтение данных из таблицы `t1`

18. у вас есть идеи почему? ведь права то дали? 

По умолчанию все объекты создаются в схеме `public` о чем и сказанно в документаци в главе [5.9.2. The Public Schema](https://www.postgresql.org/docs/13/ddl-schemas.html)
А у пользователя `testdb` есть только права на чтение из схемы `testnm`

19. посмотрите на список таблиц 

```
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
```

20. подсказка в шпаргалке под пунктом 20 

-

21. а почему так получилось с таблицей (если делали сами и без шпаргалки то может у вас все нормально) 

По умолчанию все объекты создаются в схеме `public` о чем и сказанно в документаци в главе [5.9.2. The Public Schema](https://www.postgresql.org/docs/13/ddl-schemas.html)
А у пользователя `testdb` есть только права на чтение из схемы `testnm`

22. вернитесь в базу данных testdb под пользователем postgres

`sudo -u postgres psql`

23. удалите таблицу t1 

```
\c testdb
DROP TABLE t1;
```

24.  создайте ее заново но уже с явным указанием имени схемы testnm 

`CREATE TABLE testnm.t1 (c1 INTEGER);`

```
\dt testnm.*
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 testnm | t1   | table | postgres
(1 row)
```

25. вставьте строку со значением c1=1 

INSERT INTO testnm.t1 VALUES (1);

26. зайдите под пользователем testread в базу данных testdb 

`psql -h 127.0.0.1 -U testread -d testdb -W`

27. сделайте select * from testnm.t1; 

Ничего не было выбрано из базы

28. получилось? 

нет

29. есть идеи почему? если нет - смотрите шпаргалку 

Сначала мы выдали право на `select` а уже потом создали таблицу `t1`

30. как сделать так чтобы такое больше не повторялось? если нет идей - смотрите шпаргалку 

Необходимо изменить **DEFAULT PRIVILEGES**, как описано в главе [ALTER DEFAULT PRIVILEGES](https://www.postgresql.org/docs/13/sql-alterdefaultprivileges.html)

```
\c testdb postgres; 
ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly; 
psql -h 127.0.0.1 -U testread -d testdb -W
```

31. сделайте select * from testnm.t1;

```
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
```

32. получилось?

нет

33. есть идеи почему? если нет - смотрите шпаргалку 

Идей нет :)

```
sudo -u postgres psql
\c testdb
GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
psql -h 127.0.0.1 -U testread -d testdb -W
```

34. сделайте select * from testnm.t1; 

```
testdb=> select * from testnm.t1;
 c1
----
  1
(1 row)
```

35. получилось?

да

36. ура! 
37. теперь попробуйте выполнить команду create table t2(c1 integer); insert into t2 values (2); 

```
testdb=> create table t2(c1 integer); insert into t2 values (2);
CREATE TABLE
INSERT 0 1
testdb=> \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t2   | table | testread
(1 row)
```

38. а как так? нам же никто прав на создание таблиц и insert в них под ролью readonly? 

Из объяснения по [ссылке](https://dba.stackexchange.com/questions/108975/postgresql-who-or-what-is-the-public-role)
следует, что "Public also acts as an implicit role that other roles belong to" то есть роль `public` присваивается по умолчанию всем.
"By default it gives create permission on the public schema. 
when you don't remove this all the other correct steps to create a read only user results in 
that user also being able to create new objects in the public schema and then due to ownership put data in them."
То есть с этой ролью мы можем создавать и изменять таблицы в схеме `public`.

39. есть идеи как убрать эти права? если нет - смотрите шпаргалку

```
sudo -u postgres psql
\c testdb
REVOKE ALL ON SCHEMA public FROM PUBLIC;
REVOKE ALL ON DATABASE testdb FROM PUBLIC;
```

40. если вы справились сами то расскажите что сделали и почему, если смотрели шпаргалку - объясните что сделали и почему выполнив указанные в ней команды 

См. выше

41. теперь попробуйте выполнить команду create table t3(c1 integer); insert into t3 values (2);

```
psql -h 127.0.0.1 -U testread -d testdb -W
create table t3(c1 integer); insert into t3 values (2);
ERROR:  no schema has been selected to create in
LINE 1: create table t4(c1 integer);
...

create table public.t4(c1 integer); insert into public.t4 values (2);
ERROR:  permission denied for schema public
....
```

42. расскажите что получилось и почему

В схеме `public` больше нельзя создавать таблицы, то есть у роли `PUBLIC` были отозваны все права для работы в
 схеме `public` (в `testdb`) и в базе данных `testdb`.