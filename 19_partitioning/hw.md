# Секционирование

## Подготовка

- Создадим новую VM в GCP командой:
```shell
> gcloud compute instances create gcp-vm-postgres \
        --zone=$ZONE \
        --image=$IMAGE \
        --image-project=ubuntu-os-cloud \
        --maintenance-policy=MIGRATE \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=10GB \
        --boot-disk-type=pd-ssd \
        --tags=postgres
```

Значения переменных окружения:
```shell
export IMAGE="ubuntu-2104-hirsute-v20210928"
export ZONE="europe-north1-a"
export INSTANCE_TYPE="e2-medium"
```

- Подключимся к только что созданной VM:
```shell
> gcloud compute ssh gcp-vm-postgres 
```

- Установим `PostgreSQL` версии *14*:
```shell
> sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q \
&&  sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' \
&& wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - \
&& sudo apt-get update \
&& sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-14
```

- Скачаем архив с тестовой базой данных `flights` c [Postgres Pro](https://edu.postgrespro.ru):
```shell
> wget https://edu.postgrespro.ru/demo-big.zip
...
2021-12-27 19:10:34 (10.0 MB/s) - ‘demo-big.zip’ saved [243203214/243203214]
```

- Разархивируем и скопируем скаченный архив:
```shell
> sudo apt-get -y install unzip

> ls -lah demo-big-20170815.sql
-rw-rw-r-- 1 vlad vlad 888M Jan  5  2018 demo-big-20170815.sql

> sudo -u postgres psql < demo-big-20170815.sql
```

- Проверим, что у нас появилась база `demo`:
```shell
> sudo -u postgres psql

postgres=# \l
                              List of databases
   Name    |  Owner   | Encoding | Collate |  Ctype  |   Access privileges
-----------+----------+----------+---------+---------+-----------------------
 demo      | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 postgres  | postgres | UTF8     | C.UTF-8 | C.UTF-8 |
 template0 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
 template1 | postgres | UTF8     | C.UTF-8 | C.UTF-8 | =c/postgres          +
           |          |          |         |         | postgres=CTc/postgres
```

- Подключимся к базе `demo` и посмотрим список таблиц внутри:
```shell
postgres=# \c demo

demo=# \dt
               List of relations
  Schema  |      Name       | Type  |  Owner
----------+-----------------+-------+----------
 bookings | aircrafts_data  | table | postgres
 bookings | airports_data   | table | postgres
 bookings | boarding_passes | table | postgres
 bookings | bookings        | table | postgres
 bookings | flights         | table | postgres
 bookings | seats           | table | postgres
 bookings | ticket_flights  | table | postgres
 bookings | tickets         | table | postgres
```

- Посмотрим размер базы данных (ответ: `2.6` GiB):
```shell
demo=# SELECT
    pg_statio_user_tables.schemaname AS schema,
    pg_size_pretty(sum(pg_total_relation_size(relid))) AS total_size
FROM
    pg_catalog.pg_statio_user_tables
GROUP BY
    pg_statio_user_tables.schemaname;
    
  schema  | total_size
----------+------------
 bookings | 2631 MB
```

## Секционируем одну из самых больших таблиц в базе данных

- Посмотрим размер каждой таблицы в базе:
```shell
> demo=# SELECT
   relname AS "Table",
   pg_size_pretty(pg_total_relation_size(relid)) As "Size",
   pg_size_pretty(pg_total_relation_size(relid) - pg_relation_size(relid)) as "External Size"
 FROM pg_catalog.pg_statio_user_tables 
 ORDER BY pg_total_relation_size(relid) DESC;

      Table      |  Size   | External Size
-----------------+---------+---------------
 boarding_passes | 1102 MB | 647 MB
 ticket_flights  | 871 MB  | 325 MB
 tickets         | 475 MB  | 89 MB
 bookings        | 150 MB  | 45 MB
 flights         | 32 MB   | 11 MB
 seats           | 144 kB  | 80 kB
 airports_data   | 72 kB   | 48 kB
 aircrafts_data  | 32 kB   | 24 kB
```

- Первые три таблицы `boarding_passes`, `ticket_flights` и `tickets` для секционирования нам не очень подходят, так как
колонки внутри них это в основном `id` записей (e.g. `flight_id`, `passenger_id`) или же значения колонок абсолютно разные
без какого либо диапазон (хотя это не помешала бы использовать секционирование по `HASH`'у), наиболее удачно секционирование
подходит для таблицы `bookings` по колонке `book_date` с разбивкой по месяцам:
```shell
> demo=# \d bookings
                        Table "bookings.bookings"
    Column    |           Type           | Collation | Nullable | Default
--------------+--------------------------+-----------+----------+---------
 book_ref     | character(6)             |           | not null |
 book_date    | timestamp with time zone |           | not null |
 total_amount | numeric(10,2)            |           | not null |
Indexes:
    "bookings_pkey" PRIMARY KEY, btree (book_ref)
Referenced by:
    TABLE "tickets" CONSTRAINT "tickets_book_ref_fkey" FOREIGN KEY (book_ref) REFERENCES bookings(book_ref)
```

- Прикинем, сколько секций по месяцам нам будет необходимо создать (ответ: 14'ть):
```shell
> demo=# SELECT to_char(date_trunc('month', book_date), 'YYYY-MM') AS months FROM bookings GROUP BY date_trunc('month', book_date) ORDER BY months;
 months
---------
 2016-07
 2016-08
 2016-09
 2016-10
 2016-11
 2016-12
 2017-01
 2017-02
 2017-03
 2017-04
 2017-05
 2017-06
 2017-07
 2017-08
(14 rows)
```

- Предположим, что чаще всего по данной таблице мы пытаемся найти общую сумму бронирований за какой-либо период времени:
```shell
demo=# EXPLAIN ANALYZE SELECT SUM(total_amount) FROM bookings WHERE book_date BETWEEN '2017-02-01'::date AND '2017-02-18'::date;
                                                                QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=27737.09..27737.10 rows=1 width=32) (actual time=336.804..339.452 rows=1 loops=1)
   ->  Gather  (cost=27736.87..27737.08 rows=2 width=32) (actual time=334.794..339.433 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=26736.87..26736.88 rows=1 width=32) (actual time=299.725..299.726 rows=1 loops=3)
               ->  Parallel Seq Scan on bookings  (cost=0.00..26641.44 rows=38173 width=6) (actual time=0.034..288.459 rows=31235 loops=3)
                     Filter: ((book_date >= '2017-02-01'::date) AND (book_date <= '2017-02-18'::date))
                     Rows Removed by Filter: 672468
 Planning Time: 0.738 ms
 Execution Time: 339.494 ms
```

- Запрос выполняется сейчас за 339 миллисекунд и использует `Parallel Seq Scan` по таблице `bookings`, ничего необычного

- Перед созданием секций, проверим значение параметра `enable_partition_pruning`:
```shell
demo=# show enable_partition_pruning;
 enable_partition_pruning
--------------------------
 on
```

> enable_partition_pruning (boolean)
> Включает или отключает в планировщике возможность устранять секции секционированных таблиц из планов запроса. 
> Также влияет на возможность планировщика генерировать планы запросов, позволяющие исполнителю пропускать 
> (игнорировать) секции при выполнении запросов.

- Создадим секционированную таблицу `bookings_partition`, в которую впоследствии вставим данные из оригинальной таблице 
`bookings`. Эта таблица будет полной копией оригинальной (кроме ограничения `tickets_book_ref_fkey`, он нам не нужен в 
- данный момент):
```shell
> demo=# CREATE TABLE bookings.bookings_partition (
    book_ref character(6) NOT NULL,
    book_date timestamp with time zone NOT NULL,
    total_amount numeric(10,2) NOT NULL
) PARTITION BY RANGE(book_date);
```

- :warning: Данные в созданную таблицу `bookings_partition` необходимо вставить, только после создания секций, чтобы
строки смогли распределиться по секциям

- Создадим 14-ть секций для каждого месяца:
```shell
CREATE TABLE bookings.bookings_partition_2016_07 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-07-01') TO ('2016-08-01');
CREATE TABLE bookings.bookings_partition_2016_08 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-08-01') TO ('2016-09-01');
CREATE TABLE bookings.bookings_partition_2016_09 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-09-01') TO ('2016-10-01');
CREATE TABLE bookings.bookings_partition_2016_10 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-10-01') TO ('2016-11-01');
CREATE TABLE bookings.bookings_partition_2016_11 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-11-01') TO ('2016-12-01');
CREATE TABLE bookings.bookings_partition_2016_12 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2016-12-01') TO ('2017-01-01');
CREATE TABLE bookings.bookings_partition_2017_01 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-01-01') TO ('2017-02-01');
CREATE TABLE bookings.bookings_partition_2017_02 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-02-01') TO ('2017-03-01');
CREATE TABLE bookings.bookings_partition_2017_03 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-03-01') TO ('2017-04-01');
CREATE TABLE bookings.bookings_partition_2017_04 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-04-01') TO ('2017-05-01');
CREATE TABLE bookings.bookings_partition_2017_05 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-05-01') TO ('2017-06-01');
CREATE TABLE bookings.bookings_partition_2017_06 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-06-01') TO ('2017-07-01');
CREATE TABLE bookings.bookings_partition_2017_07 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-07-01') TO ('2017-08-01');
CREATE TABLE bookings.bookings_partition_2017_08 PARTITION OF bookings.bookings_partition FOR VALUES FROM ('2017-08-01') TO ('2017-09-01');
```

- А вот сейчас мы можем скопировать все строки из оригинальной таблицы:
```shell
demo=# INSERT INTO bookings.bookings_partition SELECT * FROM bookings.bookings;
INSERT 0 2111110
```

- Проверим, что данные были добавлены и посмотрим максимальное и минимальное значение в секции:
```shell
demo=# SELECT * FROM bookings_partition_2017_02 LIMIT 10;
 book_ref |       book_date        | total_amount
----------+------------------------+--------------
 0000E2   | 2017-02-13 17:45:00+00 |    112000.00
 000146   | 2017-02-01 09:02:00+00 |     48800.00
 00017E   | 2017-02-19 21:48:00+00 |    174400.00
 000207   | 2017-02-06 06:22:00+00 |    128800.00
 000258   | 2017-02-04 07:38:00+00 |     25200.00
 0002B5   | 2017-02-24 10:23:00+00 |    209900.00
 0005F7   | 2017-02-05 01:50:00+00 |    257500.00
 00061E   | 2017-02-21 12:59:00+00 |     62300.00
 000695   | 2017-02-08 18:40:00+00 |     62600.00
 0006E2   | 2017-02-15 19:05:00+00 |     15000.00

demo=# SELECT MAX(book_date) FROM bookings_partition_2017_02 LIMIT 10;
          max
------------------------
 2017-02-28 23:59:00+00

demo=# SELECT MIN(book_date) FROM bookings_partition_2017_02 LIMIT 10;
          min
------------------------
 2017-02-01 00:00:00+00
```

- Посмотрим на скорость выполнения и план запроса по секционированной таблице:
```shell
demo=# EXPLAIN ANALYZE SELECT SUM(total_amount) FROM bookings_partition WHERE book_date BETWEEN '2017-02-01'::date AND '2017-02-18'::date;

                                                                                       QUERY PLAN
-----------------------------------------------------------------------------------------------------------------------
 Finalize Aggregate  (cost=33376.54..33376.55 rows=1 width=32) (actual time=68.059..74.360 rows=1 loops=1)
   ->  Gather  (cost=33376.31..33376.52 rows=2 width=32) (actual time=67.819..74.346 rows=3 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Partial Aggregate  (cost=32376.31..32376.32 rows=1 width=32) (actual time=34.233..34.234 rows=1 loops=3)
               ->  Parallel Append  (cost=0.00..32278.02 rows=39316 width=6) (actual time=0.036..25.187 rows=31235 loops=3)
                     Subplans Removed: 13
                     ->  Parallel Seq Scan on bookings_partition_2017_02 bookings_partition_1  (cost=0.00..2349.10 rows=55486 width=6) (actual time=0.035..21.844 rows=31235 loops=3)
                           Filter: ((book_date >= '2017-02-01'::date) AND (book_date <= '2017-02-18'::date))
                           Rows Removed by Filter: 20297
 Planning Time: 0.299 ms
 Execution Time: 74.433 ms
```

# Вывод
- Запрос стал выполняться быстрее: с `339` миллисекунд ускорился до `74` миллисекунд
- Запрос сканирует не всю таблицу, а только необходимую секцию: было - `Parallel Seq Scan on bookings`, стало - `Parallel Seq Scan on bookings_partition_2017_02`
- Запрос сканирует меньше строк при фильтрации: было - `Rows Removed by Filter: 672468` , стало - `Rows Removed by Filter: 20297`