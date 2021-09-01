# Lecture 9: PostgreSQL Tuning

Полезные ссылки используемые при выполнении задания:

* [How to Benchmark PostgreSQL Performance Using Sysbench](https://severalnines.com/database-blog/how-benchmark-postgresql-performance-using-sysbench)
* [PGTune](https://pgtune.leopard.in.ua/#/)
* [PostgreSQL 9.6: Параллелизация последовательного чтения](https://habr.com/ru/post/305662/)

<div align="center"><h2>1. Подготовка БД К Тестированию</h2></div>

1.1. Создадим VM c большим, чем обычно диском - 50GB:
```shell
gcloud compute instances create postgres-tune \
--zone=$ZONE \
--image=$IMAGE \
--image-project=ubuntu-os-cloud \
--maintenance-policy=MIGRATE \
--machine-type=$INSTANCE_TYPE \
--boot-disk-size=50GB \
--boot-disk-type=pd-ssd
```

1.2. Установим PostgreSQL (команда такая же как и раньше), затем установим `sysbench` и проверим его версию (нужна 1.0.20):
```shell
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
sysbench --version
# sysbench 1.0.20
```

1.3. Действуем по инструкции, создадим базу и пользователя для тестирования:
```shell
sudo -u postgres psql
```

```sql
CREATE USER sbtest WITH PASSWORD 'password';
CREATE DATABASE sbtest;
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
```

1.4. Разрешим только что созданному пользователю подключаться локально с использованием пароля к базе:
```shell
sudo su -c 'echo "host    sbtest          sbtest          127.0.0.1/32            md5"  >> /etc/postgresql/13/main/pg_hba.conf'
# md5 - Проверяет пароль пользователя, производя аутентификацию SCRAM-SHA-256 или MD5.
```

1.5. Перезапустим кластер и проверим подключение:
```shell
sudo systemctl restart postgresql@13-main
```
```shell
psql -U sbtest -h 127.0.0.1 -p 5432 -W
```

1.6. Заполним базу тестовыми данными (данное действие займет порядка двух часов):
```shell
sudo sysbench \
--pgsql-host=127.0.0.1 \
--pgsql-port=5432 \
--pgsql-db=sbtest \
--db-driver=pgsql \
--pgsql-user=sbtest \
--pgsql-password=password \
--table_size=2000000 \
--tables=30 \
/usr/share/sysbench/oltp_read_write.lua \
prepare
```
Где:
`threads` - The total number of worker threads to create
`table_size` - number of rows in the test table
`tables` - number of tables (sbtest1 to sbtest30) inside database 'sbtest'
`/usr/share/sysbench/oltp_read_write.lua` - data will be prepared by script

1.7. Посмотрим размер созданных 30-ти тестовых таблиц:
```shell
psql -U sbtest -h 127.0.0.1 -p 5432 -W -c '\dt+\'
Password:
List of relations
Schema |   Name   | Type  | Owner  | Persistence | Size   | Description
--------+----------+-------+--------+-------------+--------+-------------
public | sbtest1  | table | sbtest | permanent   | 424 MB |
...
(30 rows)
```

1.8. Проверим размер всей базы (15GB):
```shell
postgres=# SELECT pg_size_pretty(pg_database_size('sbtest'));
pg_size_pretty
----------------
14642 MB
```

1.9. Посмотрим структуру тестовых таблиц:
```shell
sbtest=# \d sbtest1
Table "public.sbtest1"
Column |      Type      | Collation | Nullable |               Default
--------+----------------+-----------+----------+-------------------------------------
id     | integer        |           | not null | nextval('sbtest1_id_seq'::regclass)
k      | integer        |           | not null | 0
c      | character(120) |           | not null | ''::bpchar
pad    | character(60)  |           | not null | ''::bpchar
Indexes:
"sbtest1_pkey" PRIMARY KEY, btree (id)
"k_1" btree (k)
```

1.10. Также перед началом тестирования посмотрим, как выглядят данные в наших тестовых таблицах:
```shell
sbtest=# select * from sbtest1 limit 1 \gx
-[ RECORD 1 ]-----------------------------------------------------------------------------------------------------------------
id  | 1
k   | 49929
c   | 83868641912-28773972837-60736120486-75162659906-27563526494-20381887404-41576422241-93426793964-56405065102-33518432330
pad | 67847967377-48000963322-62604785301-91415491898-96926520291
```

<div align="center"><h2>2. Запуск OLTP Read/Write Нагрузки</h2></div>

2.1. Посмотрим доступные параметры для запуска теста:
```shell
sysbench /usr/share/sysbench/oltp_read_write.lua help

sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)
oltp_read_write.lua options:
--auto_inc[=on|off]           Use AUTO_INCREMENT column as Primary Key (for MySQL), or its alternatives in other DBMS. When disabled, use client-generated IDs [on]
--create_secondary[=on|off]   Create a secondary index in addition to the PRIMARY KEY [on]
--delete_inserts=N            Number of DELETE/INSERT combinations per transaction [1]
--distinct_ranges=N           Number of SELECT DISTINCT queries per transaction [1]
--index_updates=N             Number of UPDATE index queries per transaction [1]
--mysql_storage_engine=STRING Storage engine, if MySQL is used [innodb]
--non_index_updates=N         Number of UPDATE non-index queries per transaction [1]
--order_ranges=N              Number of SELECT ORDER BY queries per transaction [1]
--pgsql_variant=STRING        Use this PostgreSQL variant when running with the PostgreSQL driver. The only currently supported variant is 'redshift'. When enabled, create_secondary is automatically disabled, and delete_inserts is set to 0
--point_selects=N             Number of point SELECT queries per transaction [10]
--range_selects[=on|off]      Enable/disable all range SELECT queries [on]
--range_size=N                Range size for range SELECT queries [100]
--secondary[=on|off]          Use a secondary index in place of the PRIMARY KEY [off]
--simple_ranges=N             Number of simple range SELECT queries per transaction [1]
--skip_trx[=on|off]           Don't start explicit transactions and execute all queries in the AUTOCOMMIT mode [off]
--sum_ranges=N                Number of SELECT SUM() queries per transaction [1]
--table_size=N                Number of rows per table [10000]
--tables=N                    Number of tables [1]
```

2.2. Запустим первый тест, выполняться он будет 10-ть минут, 64-ре потока будут посылать запросы к тестовым таблицам:
```shell
sudo sysbench \
--pgsql-host=127.0.0.1 \
--pgsql-port=5432 \
--pgsql-db=sbtest \
--db-driver=pgsql \
--pgsql-user=sbtest \
--pgsql-password=password \
--table_size=2000000 \
--tables=30 \
--report-interval=10 \
--threads=64 \
--time=600 \
--verbosity=4 \
/usr/share/sysbench/oltp_read_write.lua \
run
```

2.3. Посмотрим TPS который удалось достичь:
```shell
transactions:                        141654 (235.83 per sec.)
```

2.4. Посмотрим значения по-умолчанию для параметров, которые мы собираемся менять:
```sql
SELECT name, setting, vartype, unit FROM pg_settings WHERE name IN (
'max_connections',
'shared_buffers ',
'effective_cache_size',
'maintenance_work_mem',
'checkpoint_completion_target',
'wal_buffers',
'default_statistics_target',
'random_page_cost',
'effective_io_concurrency',
'work_mem',
'min_wal_size',
'max_wal_size',
'max_worker_processes',
'max_parallel_workers_per_gather',
'max_parallel_workers',
'max_parallel_maintenance_workers'
);

               name               | setting
----------------------------------+---------
 max_connections                  | 100
 shared_buffers                   | 16384 x 8kB = 128MB
 effective_cache_size             | 524288 x 8kB = 4GB
 maintenance_work_mem             | 65536kB = 64MB
 checkpoint_completion_target     | 0.5
 wal_buffers                      | 512 x 8kB = 4MB
 default_statistics_target        | 100
 random_page_cost                 | 4
 effective_io_concurrency         | 1
 work_mem                         | 4096kB = 4MB
 min_wal_size                     | 80MB
 max_wal_size                     | 1024MB = 1GB
 max_worker_processes             | 8
 max_parallel_workers_per_gather  | 2
 max_parallel_workers             | 8
 max_parallel_maintenance_workers | 2
```

2.5. Скопируем рекомендуемые значения предложенные сервисом `PGTune` для `Online transaction processing system`:
```shell
sudo tee -a /etc/postgresql/13/main/postgresql.conf > /dev/null <<EOT
max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 256MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 16MB
min_wal_size = 2GB 
max_wal_size = 8GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
EOT
```

2.6. Сравним значения относительно значений по-умолчанию:

| Property | Default Value | PG Tune Read/Write | PG Tune Read-Only |
|:-----------------------------------|:------|:------|:------|
| `max_connections`                  | 100   | 100   | 100   |
| `shared_buffers`                   | 128MB | 1GB   | 1GB   |
| `effective_cache_size`             | 4GB   | 3GB   | 3GB   |
| `maintenance_work_mem`             | 64MB  | 256MB | 512MB |
| `checkpoint_completion_target`     | 0.5   | 0.9   | 0.9   |
| `wal_buffers`                      | 4MB   | 16MB  | 16MB  |
| `default_statistics_target`        | 100   | 100   | 500   |
| `random_page_cost`                 | 4     | 1.1   | 1.1   |
| `effective_io_concurrency`         | 1     | 200   | 200   |
| `work_mem`                         | 4MB   | 16MB  | 8MB   |
| `min_wal_size`                     | 80MB  | 2GB   | 4GB   |
| `max_wal_size`                     | 1GB   | 8GB   | 16GB  |
| `max_worker_processes`             | 8     | 2     | 2     |
| `max_parallel_workers_per_gather`  | 2     | 1     | 1     |
| `max_parallel_workers`             | 8     | 2     | 2     |
| `max_parallel_maintenance_workers` | 2     | 1     | 1     |

2.7. Смысл каждого параметра:

| Property | Description |
|:-----------------------------------|:--------------------------------------------------------------------------------|
| `max_connections`                  | Количество подключений, threads=64 + 3 несколько подключений для самого PostgreSQL
| `shared_buffers`                   | Используется для кэширования данных. 25% от общей оперативной памяти на сервере
| `effective_cache_size`             | Служит подсказкой для планировщика запросов, сколько RAM у него в запасе
| `maintenance_work_mem`             | Определяет максимальное количество RAM для операций типа `VACUUM`, `CREATE INDEX`, `CREATE FOREIGN KEY`
| `checkpoint_completion_target`     | Коэффициент для общего времени между контрольными точками, чем реже, тем дольше будет восстановление БД после сбоя
| `wal_buffers`                      | Объём разделяемой памяти, который будет использоваться для буферизации данных WAL
| `default_statistics_target`        | Устанавливает целевое ограничение статистики, чем больше, тем дольше выполняется `ANALYZE`
| `random_page_cost`                 | Произвольный доступ к диску дороже последовательного доступа, для HDD в 4-ре раза, для SSD в 1.1
| `effective_io_concurrency`         | Допустимое число одновременных параллельных операций I/O, для SSD несколько сотен
| `work_mem`                         | Используется для сортировок, построения hash таблиц при выполнения запроса
| `min_wal_size`                     | Пока WAL занимает на диске меньше этого объёма, старые файлы в контрольных точках перерабатываются, а не удаляются
| `max_wal_size`                     | Максимальный размер, до которого может вырастать WAL между автоматическими контрольными точками в WAL
| `max_worker_processes`             | Максимальное число фоновых процессов
| `max_parallel_workers_per_gather`  | Количество воркеров, которые могут участвовать в последовательном сканировании таблицы
| `max_parallel_workers`             | Количество воркеров, которое система сможет поддерживать для параллельных запросов
| `max_parallel_maintenance_workers` | Количество воркеров, которые могут запускаться одной служебной командой

2.8. Применим новые настройки:
```shell
sudo pg_ctlcluster 13 main restart
```

2.9. Запустим тест еще раз:
```shell
sudo sysbench \
--pgsql-host=127.0.0.1 \
--pgsql-port=5432 \
--pgsql-db=sbtest \
--db-driver=pgsql \
--pgsql-user=sbtest \
--pgsql-password=password \
--table_size=2000000 \
--tables=30 \
--report-interval=10 \
--threads=64 \
--time=600 \
--verbosity=4 \
/usr/share/sysbench/oltp_read_write.lua \
run
```

2.10. Посмотрим TPS который удалось достичь с новыми настройками:
```shell
transactions:                        128447 (213.22 per sec.)
```

2.11. Удалим настройки и перезапустим кластер

<div align="center"><h2>3. Запуск OLTP Read-Only Нагрузки</h2></div>

3.1. Запустим тест указав `oltp_read_only.lua` скрипт:
```shell
sudo sysbench \
--pgsql-host=127.0.0.1 \
--pgsql-port=5432 \
--pgsql-db=sbtest \
--db-driver=pgsql \
--pgsql-user=sbtest \
--pgsql-password=password \
--table_size=2000000 \
--tables=30 \
--report-interval=10 \
--threads=64 \
--time=600 \
--verbosity=4 \
/usr/share/sysbench/oltp_read_only.lua \
run
```

3.2. Посмотрим TPS который удалось достичь:
```shell
transactions:                        158745 (264.23 per sec.)
```

3.3. Скопируем рекомендуемые значения предложенные сервисом `PGTune` для `Data warehouse`:
```shell
sudo tee -a /etc/postgresql/13/main/postgresql.conf > /dev/null <<EOT
max_connections = 100
shared_buffers = 1GB
effective_cache_size = 3GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 16MB
default_statistics_target = 500
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 8MB
min_wal_size = 4GB
max_wal_size = 16GB
max_worker_processes = 2
max_parallel_workers_per_gather = 1
max_parallel_workers = 2
max_parallel_maintenance_workers = 1
EOT
```

3.4. Сравним значения предложенные `PG Tune` со значениями по-умолчанию:

| Property | Default Value | PG Tune Value  |
|:-----------------------------------|:------|:------|
| `max_connections`                  | 100   | 100   |
| `shared_buffers`                   | 128MB | 1GB   |
| `effective_cache_size`             | 4GB   | 3GB   |
| `maintenance_work_mem`             | 64MB  | 512MB |   
| `checkpoint_completion_target`     | 0.5   | 0.9   |
| `wal_buffers`                      | 4MB   | 16MB  |
| `default_statistics_target`        | 100   | 500   |
| `random_page_cost`                 | 4     | 1.1   |
| `effective_io_concurrency`         | 1     | 200   |
| `work_mem`                         | 4MB   | 8MB   |
| `min_wal_size`                     | 80MB  | 4GB   |
| `max_wal_size`                     | 1GB   | 16GB  |
| `max_worker_processes`             | 8     | 2     |
| `max_parallel_workers_per_gather`  | 2     | 1     |
| `max_parallel_workers`             | 8     | 2     |
| `max_parallel_maintenance_workers` | 2     | 1     |

3.5. Применим новые настройки:
```shell
sudo pg_ctlcluster 13 main restart
```

3.6. Запустим тест еще раз:
```shell
sudo sysbench \
--pgsql-host=127.0.0.1 \
--pgsql-port=5432 \
--pgsql-db=sbtest \
--db-driver=pgsql \
--pgsql-user=sbtest \
--pgsql-password=password \
--table_size=2000000 \
--tables=30 \
--report-interval=10 \
--threads=64 \
--time=600 \
--verbosity=4 \
/usr/share/sysbench/oltp_read_only.lua \
run
```

3.6. Посмотрим TPS:
```shell
transactions:                        193423 (321.62 per sec.)
```

<div align="center"><h2>4. Итоги</h2></div>

| Test | PG_Tune DB Type | TPS |
|:----------------------|:-------------------------------------|:-------------------|
| `oltp_read_write.lua` | -                                    | 141 654            |
| `oltp_read_write.lua` | Online transaction processing system | 128 447            |
| `oltp_read_only.lua`  | -                                    | 158 745            |
| `oltp_read_only.lua`  | Data warehouse                       | 193 423            |

* В первом тесте рекомендуемые `PG Tune` настройки привели к снижению производительности, скорее всего на данных 
  размерах базы и длительности теста использование большего размера `work_mem` и `shared_buffers` менее выгодно;
* Во втором же тесте производительность выросла ~22%, что говорит о том, что даже базовый тюнинг может привести к 
  существенному росту производительности.