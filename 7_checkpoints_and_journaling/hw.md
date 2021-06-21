# Seventh Homework - Checkpoints and Journaling

### 1. Настройте выполнение контрольной точки раз в 30 секунд.

Установим значение `checkpoint_timeout` в файле `postgresql.conf` равным "30s":
```
echo 'checkpoint_timeout = 30s' >> /etc/postgresql/13/main/postgresql.conf
```

Зайдем в `psql` перечитаем конфиг и убедимся, что значение установлено:
```
SELECT pg_reload_conf();
postgres=# SELECT name, setting||' '||unit as value, context, short_desc FROM pg_settings WHERE name = 'checkpoint_timeout';
        name        | value | context |                        short_desc
--------------------+-------+---------+----------------------------------------------------------
 checkpoint_timeout | 30 s  | sighup  | Sets the maximum time between automatic WAL checkpoints.
```

Также включим `log_checkpoints`, чтобы проверить, когда будет происходить выполнение контрольной точки:

> Параметр `log_checkpoints` (выключенный по умолчанию) позволяет получать в журнале сообщений сервера информацию о выполняемых контрольных точках.

```
postgres=# ALTER SYSTEM SET log_checkpoints = on;
postgres=# SELECT pg_reload_conf();
```

### 2. 10 минут c помощью утилиты pgbench подавайте нагрузку.

```
vlad@postgres-journals:~$ sudo -u postgres pgbench -c 8 -P 60 -T 600 -U postgres postgres
starting vacuum...end.
progress: 60.0 s, 787.8 tps, lat 10.148 ms stddev 8.329
progress: 120.0 s, 767.5 tps, lat 10.423 ms stddev 8.738
progress: 180.0 s, 799.7 tps, lat 10.003 ms stddev 7.986
progress: 240.0 s, 784.0 tps, lat 10.203 ms stddev 7.826
progress: 300.0 s, 792.4 tps, lat 10.095 ms stddev 7.296
progress: 360.0 s, 803.6 tps, lat 9.955 ms stddev 7.162
progress: 420.0 s, 773.8 tps, lat 10.337 ms stddev 8.515
progress: 480.0 s, 786.8 tps, lat 10.167 ms stddev 8.491
progress: 540.0 s, 792.7 tps, lat 10.092 ms stddev 8.067
progress: 600.0 s, 787.3 tps, lat 10.160 ms stddev 8.804
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 472550
latency average = 10.157 ms
latency stddev = 8.135 ms
tps = 787.564048 (including connections establishing)
tps = 787.567585 (excluding connections establishing)
```

### 3. Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходиться в среднем на одну контрольную точку.

Перед запуском теста я запустил команду которая раз в 30 секунд записывала состояние директории с wal файлами в файл `wal_files`:  

```
watch -n 30 "ls -lah /var/lib/postgresql/13/main/pg_wal | tee -a ~/wal_files"
```

Результат получился следующий:

```
root@postgres-journals:/var/lib/postgresql/13/main/pg_wal# cat ~/wal_files

total 17M
drwx------  3 postgres postgres 4.0K Jun 17 15:16 .
drwx------ 19 postgres postgres 4.0K Jun 17 15:00 ..
-rw-------  1 postgres postgres  16M Jun 17 15:17 000000010000000000000042
drwx------  2 postgres postgres 4.0K Jun  6 16:34 archive_status
total 65M
drwx------  3 postgres postgres 4.0K Jun 17 15:18 .
drwx------ 19 postgres postgres 4.0K Jun 17 15:00 ..
-rw-------  1 postgres postgres  16M Jun 17 15:18 000000010000000000000042
-rw-------  1 postgres postgres  16M Jun 17 15:18 000000010000000000000043
-rw-------  1 postgres postgres  16M Jun 17 15:18 000000010000000000000044
-rw-------  1 postgres postgres  16M Jun 17 15:18 000000010000000000000045
drwx------  2 postgres postgres 4.0K Jun  6 16:34 archive_status
total 49M
drwx------  3 postgres postgres 4.0K Jun 17 15:19 .
drwx------ 19 postgres postgres 4.0K Jun 17 15:00 ..
-rw-------  1 postgres postgres  16M Jun 17 15:18 000000010000000000000044
-rw-------  1 postgres postgres  16M Jun 17 15:19 000000010000000000000045
-rw-------  1 postgres postgres  16M Jun 17 15:19 000000010000000000000046
drwx------  2 postgres postgres 4.0K Jun  6 16:34 archive_status
...
total 49M
drwx------  3 postgres postgres 4.0K Jun 17 15:28 .
drwx------ 19 postgres postgres 4.0K Jun 17 15:00 ..
-rw-------  1 postgres postgres  16M Jun 17 15:27 000000010000000000000060
-rw-------  1 postgres postgres  16M Jun 17 15:28 000000010000000000000061
-rw-------  1 postgres postgres  16M Jun 17 15:28 000000010000000000000062
drwx------  2 postgres postgres 4.0K Jun  6 16:34 archive_status
```


```
 max_wal_size                  | 1024 MB    | sighup     | Sets the WAL size that triggers a checkpoint.
 min_wal_size                  | 80 MB      | sighup     | Sets the minimum size to shrink the WAL to.
 wal_segment_size              | 16 MB      | internal   | Shows the size of write ahead log segments.
```

Пояснение: каждый файл журнала равен 16MB, что соответствует значению параметра - `wal_segment_size`, 
каждые 30 секунд файлы удалялись, когда происходило выполнение контрольной точки, если размер файла превышал 80MB - min_wal_size.

Если задать параметр `checkpoint_timeout` большим, чем время выполнения теста (скажем - 3000s, при длительности тесте в 
30-ть минут), можно убедиться что за это время будет сгенерировано 321MB журналов и не один из них не был удален:
```
vlad@postgres-journals:~$ sudo ls -lah /var/lib/postgresql/13/main/pg_wal
total 321M
drwx------  3 postgres postgres 4.0K Jun 19 10:40 .
drwx------ 19 postgres postgres 4.0K Jun 19 09:53 ..
-rw-------  1 postgres postgres  16M Jun 19 10:18 00000001000000000000006B
-rw-------  1 postgres postgres  16M Jun 19 10:18 00000001000000000000006C
-rw-------  1 postgres postgres  16M Jun 19 10:19 00000001000000000000006D
-rw-------  1 postgres postgres  16M Jun 19 10:20 00000001000000000000006E
-rw-------  1 postgres postgres  16M Jun 19 10:21 00000001000000000000006F
-rw-------  1 postgres postgres  16M Jun 19 10:30 000000010000000000000070
-rw-------  1 postgres postgres  16M Jun 19 10:31 000000010000000000000071
-rw-------  1 postgres postgres  16M Jun 19 10:31 000000010000000000000072
-rw-------  1 postgres postgres  16M Jun 19 10:32 000000010000000000000073
-rw-------  1 postgres postgres  16M Jun 19 10:33 000000010000000000000074
-rw-------  1 postgres postgres  16M Jun 19 10:34 000000010000000000000075
-rw-------  1 postgres postgres  16M Jun 19 10:34 000000010000000000000076
-rw-------  1 postgres postgres  16M Jun 19 10:35 000000010000000000000077
-rw-------  1 postgres postgres  16M Jun 19 10:36 000000010000000000000078
-rw-------  1 postgres postgres  16M Jun 19 10:37 000000010000000000000079
-rw-------  1 postgres postgres  16M Jun 19 10:38 00000001000000000000007A
-rw-------  1 postgres postgres  16M Jun 19 10:38 00000001000000000000007B
-rw-------  1 postgres postgres  16M Jun 19 10:39 00000001000000000000007C
-rw-------  1 postgres postgres  16M Jun 19 10:40 00000001000000000000007D
-rw-------  1 postgres postgres  16M Jun 19 10:41 00000001000000000000007E
drwx------  2 postgres postgres 4.0K Jun  6 16:34 archive_status
```

### 4. Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?

Все точки были выполнены по расписанию, судя по логам:

```
root@postgres-journals:/var/lib/postgresql/13/main# cat /var/log/postgresql/postgresql-13-main.log|grep "checkpoint starting"
2021-06-17 15:00:34.587 UTC [636] LOG:  checkpoint starting: immediate force wait
2021-06-17 15:16:05.966 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:17:05.810 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:18:35.141 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:19:05.064 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:19:35.070 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:20:05.015 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:20:35.120 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:21:05.043 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:21:35.118 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:22:05.068 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:22:35.135 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:23:05.023 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:23:35.008 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:24:05.006 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:24:35.072 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:25:05.073 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:25:35.109 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:26:05.122 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:26:35.150 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:27:05.016 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:27:35.128 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:28:05.098 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:28:35.813 UTC [636] LOG:  checkpoint starting: time
2021-06-17 15:29:05.319 UTC [636] LOG:  checkpoint starting: time
```

То же самое говорит значение параметра `checkpoints_timed` из таблицы `pg_stat_bgwriter`:
```
postgres=# SELECT * FROM pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 40
checkpoints_req       | 0
checkpoint_write_time | 236052
checkpoint_sync_time  | 3916
buffers_checkpoint    | 250
buffers_clean         | 93879
maxwritten_clean      | 0
buffers_backend       | 401673
buffers_backend_fsync | 0
buffers_alloc         | 793566
stats_reset           | 2021-06-17 15:16:43.748255+00
```

> Здесь, в числе прочего, мы видим количество выполненных контрольных точек:
> - checkpoints_timed — по расписанию (по достижению checkpoint_timeout),
> - checkpoints_req — по требованию (в том числе по достижению max_wal_size).

Примечание: "по требования" выполнение контрольной точки не происходило так как не был превышен `max_wal_size`.

### 5. Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.

Создадим виртуалку с HDD (`boot-disk-type=pd-standard`):
```
gcloud compute instances create postgres-journals-hdd \
        --zone=$ZONE \
        --image=$IMAGE \
        --image-project=ubuntu-os-cloud \
        --maintenance-policy=MIGRATE \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=10GB \
        --boot-disk-type=pd-standard
```

> Запись журнала происходит в одном из двух режимов:
> - синхронном — при фиксации транзакции продолжение работы невозможно до тех пор, пока все журнальные записи об этой транзакции не окажутся на диске;
> - асинхронном — транзакция завершается немедленно, а журнал записывается в фоновом режиме.
> Синхронный режим определяется параметром `synchronous_commit` и включен по умолчанию.

Проверим, что значение параметра `synchronous_commit` имеет значение `on`:
```
postgres=# SELECT name, setting, context, short_desc FROM pg_settings WHERE name = 'synchronous_commit';
        name        | setting | context |                      short_desc
--------------------+---------+---------+-------------------------------------------------------
 synchronous_commit | on      | user    | Sets the current transaction's synchronization level.
```

Запустим тест:
```
postgres@postgres-container:/home/vlad$ pgbench -c 8 -P 60 -T 600 -U postgres postgres
starting vacuum...end.
progress: 60.0 s, 622.5 tps, lat 12.843 ms stddev 6.537
progress: 120.0 s, 618.7 tps, lat 12.928 ms stddev 8.075
progress: 180.0 s, 617.9 tps, lat 12.945 ms stddev 9.174
progress: 240.0 s, 624.3 tps, lat 12.812 ms stddev 9.253
progress: 300.0 s, 627.8 tps, lat 12.742 ms stddev 9.273
progress: 360.0 s, 623.5 tps, lat 12.828 ms stddev 8.954
progress: 420.0 s, 607.1 tps, lat 13.177 ms stddev 8.757
progress: 480.0 s, 605.8 tps, lat 13.204 ms stddev 9.398
progress: 540.0 s, 609.2 tps, lat 13.129 ms stddev 9.422
progress: 600.0 s, 615.3 tps, lat 13.001 ms stddev 9.439
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 370330
latency average = 12.959 ms
latency stddev = 8.869 ms
tps = 617.190554 (including connections establishing)
tps = 617.193471 (excluding connections establishing)
```

Результат: ~617 TPS

Выключим синхронную запись журнала: 
```
echo 'synchronous_commit = off' >> /etc/postgresql/13/main/postgresql.conf
```

И проверим, что значение действительно поменялось:
```
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
postgres=# SELECT name, setting, unit, context, short_desc FROM pg_settings WHERE name = 'synchronous_commit';
name        | setting | unit | context |                      short_desc
--------------------+---------+------+---------+-------------------------------------------------------
synchronous_commit | off     |      | user    | Sets the current transaction's synchronization level.
```

Запустим тест повторно:
```
postgres@postgres-container:/home/vlad$ pgbench -i postgres
dropping old tables...
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.09 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.41 s (drop tables 0.01 s, create tables 0.00 s, client-side generate 0.26 s, vacuum 0.07 s, primary keys 0.07 s).
postgres@postgres-container:/home/vlad$ pgbench -c 8 -P 60 -T 600 -U postgres postgres
starting vacuum...end.
progress: 60.0 s, 1932.6 tps, lat 4.137 ms stddev 1.354
progress: 120.0 s, 1915.3 tps, lat 4.176 ms stddev 1.391
progress: 180.0 s, 937.6 tps, lat 8.527 ms stddev 23.542
progress: 240.0 s, 942.5 tps, lat 8.480 ms stddev 23.541
progress: 300.0 s, 946.6 tps, lat 8.448 ms stddev 23.476
progress: 360.0 s, 949.7 tps, lat 8.423 ms stddev 23.322
progress: 420.0 s, 956.0 tps, lat 8.368 ms stddev 23.311
progress: 480.0 s, 955.0 tps, lat 8.372 ms stddev 23.251
progress: 540.0 s, 956.2 tps, lat 8.363 ms stddev 23.268
progress: 600.0 s, 958.9 tps, lat 8.342 ms stddev 23.312
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
duration: 600 s
number of transactions actually processed: 687034
latency average = 6.984 ms
latency stddev = 19.171 ms
tps = 1145.021954 (including connections establishing)
tps = 1145.027381 (excluding connections establishing)
```

Результат с асинхронной записью: ~1145 TPS, что практически в два раза превышает результат с синхронной записью.

### 6. Создайте новый кластер с включенной контрольной суммой страниц.

```
postgres=# SHOW data_checksums;
 data_checksums
----------------
 off
```

```
sudo systemctl stop postgresql@13-main.service
```

```
/usr/lib/postgresql/13/bin/pg_checksums --enable -D "/var/lib/postgresql/13/main"
```

```
postgres=# SHOW data_checksums;
 data_checksums
----------------
 on
```

- Создайте таблицу. Вставьте несколько значений.

```
postgres=# CREATE TABLE tester (id SERIAL, t TEXT);
postgres=# INSERT INTO tester(t) SELECT md5(RANDOM()::TEXT) FROM generate_series(1,500);
```

- Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло?

```
sudo systemctl stop postgresql@13-main.service
```

Найдем, где располагается файл для "отношения" нашей таблицы:
```
postgres=# SELECT pg_relation_filepath('tester');
 pg_relation_filepath
----------------------
 base/13445/16588
```

Поменяем байты:
```
sudo dd if=/dev/zero of=/var/lib/postgresql/13/main/base/13445/16588 oflag=dsync conv=notrunc bs=1 count=8
8+0 records in
8+0 records out
8 bytes copied, 0.00603808 s, 1.3 kB/s
```

```
sudo systemctl start postgresql@13-main.service
```

```
postgres=# SELECT * FROM tester;
WARNING:  page verification failed, calculated checksum 30662 but expected 53524
ERROR:  invalid page in block 0 of relation base/13445/16588
```

Пояснение: не удалось прочитать записи из таблицы, так как файл нашего "отношения" содержит поврежденный блок.

- Как проигнорировать ошибку и продолжить работу?

Необходимо отключить проверку контрольной суммы:
```
SET ignore_checksum_failure = on;
SET
postgres=# SELECT * FROM tester;
WARNING:  page verification failed, calculated checksum 30662 but expected 53524
 id  |                t
-----+----------------------------------
   1 | 1a1aac8fe348c564229d32390fbe3f10
   2 | 6e98cbd87f0cbc15dd06b5a4aafb2a19
...
```