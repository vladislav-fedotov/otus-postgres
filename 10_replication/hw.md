# Виды и устройство репликации в PostgreSQL. Практика применения

## Задание

Цель:

* Реализовать свой миникластер на трех VM. На VM1 создаем таблицы `test` для записи, `test2` для запросов на чтение.
  Создаем публикацию таблицы `test` и подписываемся на публикацию таблицы `test2` с VM2. На VM2 создаем таблицы `test2`
  для записи, `test` для запросов на чтение. Создаем публикацию таблицы `test2` и подписываемся на публикацию
  таблицы `test` с VM1. VM3 использовать как реплику для чтения и бэкапов (подписаться на таблицы из VM1 и VM2).
  Небольшое описание, того, что получилось.

* Реализовать горячее реплицирование для высокой доступности на VN4. Источником должна выступать VM3. Написать с какими
  проблемами столкнулись.

### Подготовка

Кластер для простоты развернем локально, воспользовавшись `docker-compose`:

```yaml
services:
  vm1:
    container_name: vm1
    image: sameersbn/postgresql:12-20200524
  restart: always # чтобы при перезапуске конфигурации контейнер сам поднимался
    ports:
      - "5432:5432"
  vm2:
    container_name: vm2
    image: sameersbn/postgresql:12-20200524
  restart: always
    ports:
      - "5433:5432"
  vm3:
    container_name: vm3
    image: sameersbn/postgresql:12-20200524
  restart: always
    ports:
      - "5434:5432"
  vm4:
    container_name: vm4
    image: sameersbn/postgresql:12-20200524
  restart: always
    ports:
      - "5435:5432"
```

Доставим утилиты nano - для редактирования конфигов, netcat и ping - для проверки видимости между контейнерами:

```bash
> apt-get update && apt-get install -y iputils-ping netcat nano
> ping vm2
> nc -vz vm2 5432
> psql -U postgres -c "\l"
> psql -U postgres -c "ALTER USER postgres WITH PASSWORD 'postgres';"
```

Проверим, что мы "видим" другой контейнер:

```bash
> ping slave/master
64 bytes from vm2.postgres-replication_default (172.18.0.5): icmp_seq=1 ttl=64 time=0.150 ms
> nc -vz vm2 5432
vm2 [172.18.0.5] 5432 (postgresql) open
```

Создадим базу данных на vm1 и vm2:

```bash
vm1> sudo -u postgres psql -c "create database repl"
vm2> sudo -u postgres psql -c "create database repl"
```

### Часть 1 - Перекрестная Репликация VM1 <-> VM2

На `vm1` подготовим базу, пользователя, создадим таблицы и заполним одну из них:

```bash
postgres=# CREATE DATABASE vm1_replication;
CREATE DATABASE
postgres=# \c vm1_replication
You are now connected to database "vm1_replication" as user "postgres".
vm1_replication=# CREATE TABLE test (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm1_replication=# CREATE TABLE test2 (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm1_replication=# INSERT INTO test (hash, gender)
vm1_replication-# SELECT
vm1_replication-#   md5(RANDOM()::TEXT),
vm1_replication-#   CASE
vm1_replication-#    WHEN RANDOM() < 0.5
vm1_replication-#     THEN 'male'
vm1_replication-#     ELSE 'female'
vm1_replication-#    END
vm1_replication-#  FROM generate_series(1, 1);
INSERT 0 1
vm1_replication=# CREATE USER vm1_subscriber WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'sub';
CREATE ROLE
vm1_replication=# GRANT ALL PRIVILEGES ON DATABASE vm1_replication TO vm1_subscriber;
GRANT
vm1_replication=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO vm1_subscriber;
GRANT
vm1_replication=# GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO vm1_subscriber;
GRANT
vm1_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
vm1_replication=# SELECT * FROM test2;
 id | hash | gender
----+------+--------
(0 rows)
```

То же самое проделаем на `vm2`:

```bash
postgres=# CREATE DATABASE vm2_replication;
CREATE DATABASE
postgres=# \c vm2_replication
You are now connected to database "vm2_replication" as user "postgres".
vm2_replication=# CREATE TABLE test (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm2_replication=# CREATE TABLE test2 (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm2_replication=# INSERT INTO test2 (hash, gender)
vm2_replication-# SELECT
vm2_replication-#   md5(RANDOM()::TEXT),
vm2_replication-#   CASE
vm2_replication-#    WHEN RANDOM() < 0.5
vm2_replication-#     THEN 'male'
vm2_replication-#     ELSE 'female'
vm2_replication-#    END
vm2_replication-#  FROM generate_series(1, 1);
INSERT 0 1
vm2_replication=# CREATE USER vm2_subscriber WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'sub';
CREATE ROLE
vm2_replication=# GRANT ALL PRIVILEGES ON DATABASE vm2_replication TO vm2_subscriber;
GRANT
vm2_replication=# GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO vm2_subscriber;
GRANT
vm2_replication=# GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO vm2_subscriber;
GRANT
vm2_replication=# SELECT * FROM test;
 id | hash | gender
----+------+--------
(0 rows)

vm2_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
```

На обоих VM поменяем `wal-level` на `logical`:

```sql
vm1_replication=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM

vm2_replication=# ALTER SYSTEM SET wal_level = logical;
ALTER SYSTEM
```

Перезапустим кластеры:

```bash
sudo pg_ctlcluster 12 main restart
```

На `vm1` и `vm2` создадим публикации для таблиц `test` и `test2` соответственно:

```bash
vm1_replication=# CREATE PUBLICATION table_test_publication FOR TABLE test;
CREATE PUBLICATION
vm1_replication=# SELECT * FROM pg_publication_tables;
        pubname         | schemaname | tablename
------------------------+------------+-----------
 table_test_publication | public     | test
(1 row)

vm2_replication=#  CREATE PUBLICATION table_test2_publication FOR TABLE test2;
CREATE PUBLICATION
vm2_replication=# SELECT * FROM pg_publication_tables;
         pubname         | schemaname | tablename
-------------------------+------------+-----------
 table_test2_publication | public     | test2
(1 row)
```

На `vm1` создадим подписку на публикацию `table_test2_publication` и проверим содержимое таблицы `test2`:

```bash
vm1_replication=# CREATE SUBSCRIPTION subscription_from_vm1 CONNECTION
vm1_replication-#  'host=vm2 port=5432 user=vm2_subscriber password=sub dbname=vm2_replication'
vm1_replication-#  PUBLICATION table_test2_publication WITH (copy_data = true);
NOTICE:  created replication slot "subscription_from_vm1" on publisher
CREATE SUBSCRIPTION
vm1_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
(1 row)
```

На `vm2` создадим подписку на публикацию `table_test_publication` и проверим содержимое таблицы `test`:

```bash
vm2_replication=# CREATE SUBSCRIPTION subscription_from_vm2 CONNECTION
vm2_replication-# 'host=vm1 port=5432 user=vm1_subscriber password=sub dbname=vm1_replication'
vm2_replication-# PUBLICATION table_test_publication WITH (copy_data = true);
NOTICE:  created replication slot "subscription_from_vm2" on publisher
CREATE SUBSCRIPTION
vm2_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
(1 row)
```

На `vm1` проверим состояние подписки, на `vm2` посмотрим актуальный `lsn`:

```bash
vm1_replication=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16411
subname               | subscription_from_vm1
pid                   | 311
relid                 |
received_lsn          | 0/**1672050**
last_msg_send_time    | 2021-11-16 20:02:09.563476+00
last_msg_receipt_time | 2021-11-16 20:02:09.563979+00
latest_end_lsn        | 0/**1672050**
latest_end_time       | 2021-11-16 20:02:09.563476+00

vm2_replication=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 0/**1672050**
(1 row)
```

Видим, что значение на подписчика и публикации совпадают - `1672050`

Вставим новую строку в таблицу `test2` на `vm2` и проверим содержимое этой таблицы на `vm1`:

```bash
vm2_replication=# INSERT INTO test2(hash, gender)
...
INSERT 0 1
vm2_replication=# select * from test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
(2 rows)

vm2_replication=# SELECT pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 0/16723A8
(1 row)
vm1_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
(2 rows)

vm1_replication=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16411
subname               | subscription_from_vm1
pid                   | 311
relid                 |
received_lsn          | 0/16723A8
last_msg_send_time    | 2021-11-16 20:07:23.744339+00
last_msg_receipt_time | 2021-11-16 20:07:23.744642+00
latest_end_lsn        | 0/16723A8
latest_end_time       | 2021-11-16 20:07:23.744339+00
```

### Создадим Конфликт

Вставим строку в таблицу `test2` на `vm1` и проверим подписку:

```bash
vm1_replication=# INSERT INTO test2(hash, gender)
SELECT
  md5(RANDOM()::TEXT),
  CASE
   WHEN RANDOM() < 0.5
    THEN 'male'
    ELSE 'female'
   END
 FROM generate_series(1, 1);
INSERT 0 1
vm1_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  3 | dc548cab9cde35f264f456cb962b82c8 | male
```

Вставим строку в таблицу `test2` на `vm2` и проверим, что данная строка не пришла на `vm1` и проверим подписку:

```bash
vm2_replication=# INSERT INTO test2(hash, gender)
SELECT
  md5(RANDOM()::TEXT),
  CASE
   WHEN RANDOM() < 0.5
    THEN 'male'
    ELSE 'female'
   END
 FROM generate_series(1, 1);
INSERT 0 1
vm2_replication=# select * from test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  3 | 92c138955b83164227a539c78c013898 | female
(3 rows)

vm1_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  3 | dc548cab9cde35f264f456cb962b82c8 | male

vm1_replication=# SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+----------------------
subid                 | 16411
subname               | subscription_from_vm1
pid                   |
relid                 |
received_lsn          |
last_msg_send_time    |
last_msg_receipt_time |
latest_end_lsn        |
latest_end_time       |
```

Подписка сломалась, чтобы данные с `vm2` вновь стали поступать необходимо удалить строку вызывающую конфликт:

```bash
vm1_replication=# DELETE FROM test2 WHERE id = 3;
DELETE 1
vm1_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  3 | 92c138955b83164227a539c78c013898 | female
```

Попробуем удалить строку в таблицу `test` на `vm1` и проверим, будет ли удаление реплицировано на `vm2`:

```bash
vm1_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  2 | 6c05f3cd19262f7244850c444fbb90da | male
(2 rows)

vm1_replication=# DELETE FROM test WHERE id = 2;
DELETE 1
vm1_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
(1 row)
vm2_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
(1 row)
```

Строка пропала на `vm2`, как и ожидалось.

А теперь проверим обратное, попробуем удалить строку в таблицу `test` на `vm2` проверим, будет ли удаление реплицировано
на `vm1`:

```bash
vm1_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
(2 rows)

vm2_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
(2 rows)
vm2_replication=# DELETE FROM test WHERE id = 4;
DELETE 1
vm2_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
(1 row)

vm1_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
(2 rows)
```

Строка пропала на `vm2`, а на `vm1` она осталась.

### Часть 2 - Репликация `(VM1 <-> VM2) => VM3`

На `vm3`  создадим логическую публикацию на публикацию с `vm1`, на таблицу `test` и на `vm2`, на таблицу `test2`,
проверим, что изменения приходят из обеих подписок:

```bash
postgres=# CREATE DATABASE vm3_replication;
CREATE DATABASE
postgres=# \c vm3_replication
You are now connected to database "vm3_replication" as user "postgres".
vm3_replication=# CREATE TABLE test (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm3_replication=# CREATE TABLE test2 (id SERIAL PRIMARY KEY, hash TEXT, gender TEXT);
CREATE TABLE
vm3_replication=# CREATE SUBSCRIPTION subscription_to_test2_from_vm3 CONNECTION
vm3_replication-# 'host=vm2 port=5432 user=vm2_subscriber password=sub dbname=vm2_replication'
vm3_replication-# PUBLICATION table_test2_publication WITH (copy_data = true);
vm3_replication=# SELECT * FROM test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  3 | 92c138955b83164227a539c78c013898 | female
  4 | 1342261323a303efc203f6b1c1c3cddf | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
(4 rows)
vm3_replication=# SELECT * FROM test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
(1 row)
```

Проверим статистику репликации в
представлении [pg_stat_replication](https://postgrespro.ru/docs/postgresql/14/monitoring-stats#MONITORING-PG-STAT-REPLICATION-VIEW)
, в частности увидим имена подписок в колонке -  `application_name`, c `vm2`  - **`subscription_from_vm`**
и c `vm3` - **`subscription_to_test_from_vm3`**

> Представление `pg_stat_replication` для каждого процесса-передатчика WAL будет содержать по одной строке со статистикой о репликации на ведомый сервер, к которому подключён этот процесс. В представлении перечисляются только ведомые серверы, подключённые напрямую; информация о ведомых серверах, подключённых опосредованно, не представлена.

```bash
vm1_replication=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 6447
usesysid         | 16407
usename          | vm1_subscriber
application_name | **subscription_to_test_from_vm3**
client_addr      | 172.18.0.4
client_hostname  |
client_port      | 38642
backend_start    | 2021-11-18 19:22:40.123587+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/167A380
write_lsn        | 0/167A380
flush_lsn        | 0/167A380
replay_lsn       | 0/167A380
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-18 19:24:30.822+00
-[ RECORD 2 ]----+------------------------------
pid              | 5921
usesysid         | 16407
usename          | vm1_subscriber
application_name | **subscription_from_vm2**
client_addr      | 172.18.0.5
client_hostname  |
client_port      | 35370
backend_start    | 2021-11-18 17:23:44.341669+00
backend_xmin     |
state            | streaming
sent_lsn         | 0/167A380
write_lsn        | 0/167A380
flush_lsn        | 0/167A380
replay_lsn       | 0/167A380
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2021-11-18 19:24:30.686119+00
```

Похожую информацию можно также посмотреть в
представлении [pg_replication_slots](https://postgrespro.ru/docs/postgresql/14/view-pg-replication-slots)

> Представление `pg_replication_slots` содержит список всех слотов репликации, существующих в данный момент в кластере баз данных, а также их текущее состояние.

```bash
vm1_replication=# SELECT * FROM pg_replication_slots \gx
-[ RECORD 1 ]-------+------------------------------
slot_name           | subscription_from_vm2
plugin              | pgoutput
slot_type           | logical
datoid              | 16384
database            | vm1_replication
temporary           | f
active              | t
active_pid          | 5921
xmin                |
catalog_xmin        | 561
restart_lsn         | 0/167A348
confirmed_flush_lsn | 0/167A380
-[ RECORD 2 ]-------+------------------------------
slot_name           | subscription_to_test_from_vm3
plugin              | pgoutput
slot_type           | logical
datoid              | 16384
database            | vm1_replication
temporary           | f
active              | t
active_pid          | 6447
xmin                |
catalog_xmin        | 561
restart_lsn         | 0/167A348
confirmed_flush_lsn | 0/167A380
```

### Часть 3 - Репликация `(VM1 <-> VM2) => VM3 –> VM4`

#### Настроим Синхронную Горячую Физическую Репликацию

- Если с реплики разрешено читать, она называется hot standby, иначе — warm standby
- `synchronous_commit` = `on` транзакция считается успешной если и главный, и резервный серверы сбросили данные на диск.
  Приложение не получит подтверждение команды `COMMIT`, если данные не сохранены на двух серверах.
- `synchonous_standby_names` = `''` резервные сервера, принимающие участие в синхронной репликации, список синхронных
  резервных серверов, каждый из которых идентифицируется значением `application_name` в свойствах подключения; `'*'` -
  все
- `wal_keep_segments` = `32` измеряется в сегментах журнала длинной 16МБ; 0 - выключить, говорит сколько файлов журналов
  нужно хранить серверу, чтобы резервный сервер смог догнать главный. Данный параметр не нужен в случае синхронной
  репликации (переименован в `wal_keep_size`)
- `max_wal_senders` = `10` или любое значение ≥ 2, задаёт максимально допустимое число одновременных подключений ведомых
  серверов или клиентов потокового копирования
- `hot_standby` = `on` определяет, можно ли будет подключаться к серверу и выполнять запросы в процессе восстановления.
  Значение по умолчанию — `on`. Данный параметр нужен только ведомому серверу, когда мы скопируем данные с главного при
  помощи `pg_basebackup`

На `vm3` проверим параметры конфигурирование которых было необходимо в предыдущих версиях PostgreSQL, после версии 12
значения по умолчанию вполне достаточно:

```bash
vm3_replication=# \c vm3_replication
You are now connected to database "vm3_replication" as user "postgres".
vm3_replication=# show hot_standby;
 hot_standby
-------------
 on
(1 row)
vm3_replication=# show max_wal_senders;
 max_wal_senders
-----------------
 16
(1 row)
vm3_replication=# show wal_keep_segments;
 wal_keep_segments
-------------------
 32
(1 row)
vm3_replication=# show synchronous_commit;
 synchronous_commit
--------------------
 on
(1 row)
```

На `vm3` создадим пользователя для подключения с `vm4`:

```sql
CREATE USER repl_user WITH REPLICATION LOGIN ENCRYPTED PASSWORD 'repl_user';
```

Также нам необходимо добавить IP адрес `vm4` в файл `pg_hba.conf` на `vm3`, чтобы разрешить c нее репликацию:

```bash
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' vm4
172.18.0.4
```

> `replication` - значение `replication` в данной колонке показывает, что запись соответствует, если запрашивается
> подключение для физической репликации (имейте в виду, что для таких подключений не выбирается какая-то конкретная база данных).

```bash
> echo 'host replication repl_user 172.18.0.4/32 md5' >> /var/lib/postgresql/12/main/pg_hba.conf
> sudo pg_ctlcluster 12 main restart
```

`pg_basebackup`:

- `--progress` `-P` - включает отчёт о прогрессе
- `—-write-recovery-conf` `-R` - cоздать пустой
  файл `[standby.signal](https://postgrespro.ru/docs/postgrespro/13/warm-standby#FILE-STANDBY-SIGNAL)` (
  в `/var/lib/postgresql/12/main`) и добавить параметры конфигурации в файл `postgresql.auto.conf`, это упрощает
  настройку реплики при
  восстановлении (`primary_conninfo = 'user=repl_user passfile=''/root/.pgpass'' host=vm3 port=5432 sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any'`
  содержимое файла)
- `--wal-method=stream` `-X s` - включает в резервную копию все необходимые WAL файлы, `stream` (по умолчанию) -
  передавать WAL в процессе создания резервной копии. При выборе этого метода открывается второе соединение к серверу,
  через которое будет передаваться WAL файлы параллельно с созданием копии. Таким образом, этот метод требует
  использования не одного, а двух соединений репликации, но если клиент будет успевать получать данные журнала
  предзаписи, на исходном сервере не потребуется сохранять дополнительные журналы
- `--checkpoint=fast` `-c fast` - устанавливает режим контрольных точек: fast (быстрый) или spread (протяжённый, по
  умолчанию)

#### Почему `--wal-method=stream`:

По умолчанию `pg_basebackup` подключается к главному серверу и начинает копировать данные, но пока происходит
копирование эти данные могут модифицироваться, поэтому данные в резервной копии могут оказать несогласованными. Эту
несогласованность можно устранить в процессе восстановления с помощью журнала транзакций. Параметр `stream` позволяет
создать согласованную резервную копию, из которой можно восстановить базу данных, не воспроизводя журнал транзакций. Это
удобно, если нужно клонировать экземпляр, а не восстанавливать данные на определенный момент времени.

#### Почему `--checkpoint=fast`:

Обычно `pg_basebackup` ждет, пока главный сервер не создаст контрольную точку, так как процесс восстановления должен с
чего-то начинаться. Контрольная точка гарантирует, что записаны все данные вплоть до определенного момента, поэтому
PostgreSQL может безопасно начать восстановление с этого состояния. `pg_basebackup` может отложить начало копирования,
если интервал между контрольными точками большой, но мы хотим этого избежать и начать копирование данных, как можно
быстрее.

Итак, в контейнере с `vm4` удалим содержимое каталога `/var/lib/postgresql/12/main/*` и выполним `pg_basebackup` с
параметрами указанными выши, после выполнения, виртуальная машина перезапуститься, необходимо будет подождать несколько
секунд пока запустится Postgres, после чего зайдя в `psql` мы обнаружим базу данных `vm3_replication`, а в ней
таблицы `test` и `test2` со всем содержимым:

```bash
> docker exec -it vm4 bash -c ' \
 rm -rdf /var/lib/postgresql/12/main/* && \
 pg_basebackup \
 --progress \
 --write-recovery-conf \
 --wal-method=stream \
 --checkpoint=fast \
 --host=vm3 \
 --username=repl_user \
 --pgdata=/var/lib/postgresql/12/main'
Password:
32697/32697 kB (100%), 1/1 tablespace

❯ docker exec -it vm4 bash
root@2b2a6f55702f:/var/lib/postgresql# sudo -u postgres psql
psql (12.3 (Ubuntu 12.3-1.pgdg18.04+1))
Type "help" for help.

postgres=# \l
                                List of databases
      Name       |  Owner   | Encoding | Collate | Ctype |   Access privileges
-----------------+----------+----------+---------+-------+-----------------------
 postgres        | postgres | UTF8     | C       | C     |
 template0       | postgres | UTF8     | C       | C     | =c/postgres          +
                 |          |          |         |       | postgres=CTc/postgres
 template1       | postgres | UTF8     | C       | C     | =c/postgres          +
                 |          |          |         |       | postgres=CTc/postgres
 vm3_replication | postgres | UTF8     | C       | C     |
(4 rows)

postgres=# \c vm3_replication
You are now connected to database "vm3_replication" as user "postgres".
vm3_replication=# \dt
         List of relations
 Schema | Name  | Type  |  Owner
--------+-------+-------+----------
 public | test  | table | postgres
 public | test2 | table | postgres
(2 rows)

vm3_replication=# table test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
  5 | d43aed7b6e12af9c7329a5d02202b2c0 | male
  6 | 94995e526b0d3cd6ea88f08aac70b33c | female
(4 rows)

vm3_replication=# table test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  3 | 92c138955b83164227a539c78c013898 | female
  4 | 1342261323a303efc203f6b1c1c3cddf | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
(4 rows)
```

Проверим состояние репликации, на `vm3` посмотрим в `pg_stat_replication`:

```bash
postgres=# select usename,application_name,client_addr,backend_start,state,sync_state from pg_stat_replication;
  usename  | application_name | client_addr |         backend_start         |   state   | sync_state
-----------+------------------+-------------+-------------------------------+-----------+------------
 repl_user | walreceiver      | 172.18.0.4  | 2021-11-28 18:33:19.512637+00 | streaming | async
(1 row)
```

На `vm4` проверим представление `pg_stat_wal_receiver`:

```bash
postgres=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 203
status                | streaming
receive_start_lsn     | 0/F000000
receive_start_tli     | 1
received_lsn          | 0/F000148
received_tli          | 1
last_msg_send_time    | 2021-11-28 20:25:41.747975+00
last_msg_receipt_time | 2021-11-28 20:25:41.748474+00
latest_end_lsn        | 0/F000148
latest_end_time       | 2021-11-28 20:24:11.597373+00
slot_name             |
sender_host           | vm3
sender_port           | 5432
conninfo              | user=repl_user password=******** dbname=replication host=vm3 port=5432 fallback_application_name=walreceiver sslmode=prefer sslcompression=0 gssencmode=prefer krbsrvname=postgres target_session_attrs=any
```

Этот вызов показывает наличие одного процесса "WAL reciever" с PID - 203, который в данный момент активен и получил
данные от `0/F000000` до `0/F000148`. Timeline
ID ([TLI](https://www.postgresql.org/docs/current/static/continuous-archiving.html#BACKUP-TIMELINES)) был один во время
старта и сейчас также равен единице. Разница между временем отправки (`last_msg_send_time`) и временем получения
последнего сообщения (`last_msg_receipt_time`) означает latency подключения, в данном случае оно меньше одной
миллисекунды. Разница между двумя LSN (начальным `receive_start_lsn` - и последним `latest_end_lsn`) обозначает
количество переданных данных (`0xF000148` - `0xF000000` = 328 Bytes). Также можно заметить, что подключение не
использует слот для репликации, если бы он использовался `slot_name` не был бы null.

В завершении, чтобы убедиться, что вся цепочка репликации работает вставить по одной строке в таблицу `test` на `vm1` и
в таблицу `test2` на `vm2`:

```bash
vm1_replication=# table test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
  5 | d43aed7b6e12af9c7329a5d02202b2c0 | male
  6 | 94995e526b0d3cd6ea88f08aac70b33c | female
  7 | 5da1310e4a76412024dcefb338333cf2 | female
(5 rows)

vm2_replication=# table test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  3 | 92c138955b83164227a539c78c013898 | female
  4 | 1342261323a303efc203f6b1c1c3cddf | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  5 | 8c4443083326f7a49f2c940cf9eecd30 | female
(5 rows)
```

И проверим, что они появились на `vm3`:

```bash
vm3_replication=# table test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
  5 | d43aed7b6e12af9c7329a5d02202b2c0 | male
  6 | 94995e526b0d3cd6ea88f08aac70b33c | female
  7 | 5da1310e4a76412024dcefb338333cf2 | female
(5 rows)

vm3_replication=# table test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  3 | 92c138955b83164227a539c78c013898 | female
  4 | 1342261323a303efc203f6b1c1c3cddf | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  5 | 8c4443083326f7a49f2c940cf9eecd30 | female
(5 rows)
```

И `vm4`:

```bash
vm3_replication=# table test;
 id |               hash               | gender
----+----------------------------------+--------
  1 | fa99ed70c31ff758083e020bae3a5be5 | male
  4 | 836bb71c35566c0a8db26f7f32acacca | female
  5 | d43aed7b6e12af9c7329a5d02202b2c0 | male
  6 | 94995e526b0d3cd6ea88f08aac70b33c | female
  7 | 5da1310e4a76412024dcefb338333cf2 | female
(5 rows)

vm3_replication=# table test2;
 id |               hash               | gender
----+----------------------------------+--------
  1 | 19f3b94424dbd4f38c4f1960e3406052 | male
  3 | 92c138955b83164227a539c78c013898 | female
  4 | 1342261323a303efc203f6b1c1c3cddf | male
  2 | 7749dfb3ac23effdb6549fb3ab2e570d | female
  5 | 8c4443083326f7a49f2c940cf9eecd30 | female
(5 rows)
```

## Полезные Ссылки

[Monitoring Postgres Replication](https://pgdash.io/blog/monitoring-postgres-replication.html)
[Записки программиста](https://eax.me/postgresql-replication/)
[Логическая репликация в PostgreSQL. Репликационные идентификаторы и популярные ошибки](https://habr.com/ru/company/postgrespro/blog/489308/)
[PostgreSQL : Документация: 14: pg_basebackup](https://postgrespro.ru/docs/postgresql/14/app-pgbasebackup)
[https://www.youtube.com/watch?v=lnY9MIyiALY](https://www.youtube.com/watch?v=lnY9MIyiALY)
[Docker PostgreSQL](https://hub.docker.com/r/sameersbn/postgresql)