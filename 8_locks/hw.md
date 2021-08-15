# Eighth Homework - Locks

***
<div align="center"><h2>1. Настройка Сервера</h2></div>

***
### 1.1 Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.
Из документации:
> Для мониторинга времени блокировок Необходимо включить параметр `log_lock_waits`. 
В этом случае в журнал сообщений сервера будет попадать информация, если транзакция ждала дольше, 
чем `deadlock_timeout` (несмотря на то, что используется параметр для взаимоблокировок, речь идет об обычных ожиданиях).

Для начала посмотрим значение поля `context` из таблицы `pg_settings`, чтобы понять как нам установить значения для параметров -
`log_lock_waits` и `deadlock_timeout`:

```sql
SELECT name, setting, context, short_desc FROM pg_settings WHERE name IN ('log_lock_waits', 'deadlock_timeout');
```
```
       name       |  setting  |  context  |                          short_desc
------------------+-----------+-----------+---------------------------------------------------------------
 deadlock_timeout | 1000 ms   | superuser | Sets the time to wait on a lock before checking for deadlock.
 log_lock_waits   | off       | superuser | Logs long lock waits.
```
Из документации:
> `superuser` - эти параметры можно изменить в `postgresql.conf`, либо в рамках сеанса, командой `SET`; 
но только суперпользователи могут менять их, используя `SET`. Изменения в `postgresql.conf` будут отражены в
существующих сеансах, только если в них командой `SET` не были заданы локальные значения.

Необходимо изменить значение `deadlock_timeout`, с 1000 миллисекунд (по-умолчанию) на 200 миллисекунд и включить, то 
есть установить значение `'on'`, для `log_lock_waits`. Так как в дальнейшем будут необходимы разные параллельные 
подключения для воспроизведения различных сценариев блокировки, логичнее воспользоваться командой `ALTER SYSTEM SET`
для изменения параметров, чтобы "изменения параметров конфигурации сервера, распространилось на весь кластер баз 
данных." нежели `SET param TO 'value';`, которая установит значение только для текущей сессии.

```sql
ALTER SYSTEM SET log_lock_waits = 'on';
ALTER SYSTEM

ALTER SYSTEM SET deadlock_timeout = '200ms';
ALTER SYSTEM

SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t

SHOW log_lock_waits;
 log_lock_waits
----------------
 on

SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
```

### 1.2 Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
 
Для демонстрации создадим базу данных `locks` и подключаемся к ней:
```sql
CREATE DATABASE locks;
\c locks
```

Создадим таблицу `accounts` для демонстрации и заполним ее данными:
```sql
INSERT INTO accounts VALUES (1,1000.00), (2,2000.00), (3,3000.00);
```

В другом терминале начнем наблюдение за логами кластера postgresql (SHIFT+G и SHIFT+F для перехода в конец файла и 
наблюдением за изменением):
```shell
less  /var/log/postgresql/postgresql-13-main.log
```
```shell
2021-08-13 17:43:00.321 UTC [625] LOG:  received SIGHUP, reloading configuration files
2021-08-13 17:43:00.321 UTC [625] LOG:  parameter "log_lock_waits" changed to "on"
2021-08-13 17:43:00.321 UTC [625] LOG:  parameter "deadlock_timeout" changed to "200ms"
Waiting for data... (interrupt to abort)
```

В первом терминале (далее `(I)>`) начнем транзакцию, проверим PID и возьмем блокировку на таблицу `accounts`:
```sql
(I)> BEGIN;
BEGIN
(I)> SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1304
(I)> LOCK TABLE accounts;
LOCK TABLE
```

Во втором терминале (далее `(II)>`), также начнем транзакцию, проверим PID и попробуем взять блокировку на таблицу 
`accounts`:
```sql
(II)> BEGIN;
BEGIN
(II)> SELECT pg_backend_pid();
 pg_backend_pid
----------------
           1469
(II)> LOCK TABLE accounts; -- awating 
```

В логах увидим сообщение, о том что процесс ждет блокировку (AccessExclusiveLock) дольше 200 миллисекунд и запрос, 
которому эта блокировка необходима:
```shell
2021-08-13 18:03:40.878 UTC [1469] postgres@habr LOG:  process 1469 still waiting for AccessExclusiveLock on relation 16411 of database 16410 after 200.144 ms
2021-08-13 18:03:40.878 UTC [1469] postgres@habr DETAIL:  Process holding the lock: 1304. Wait queue: 1469.
2021-08-13 18:03:40.878 UTC [1469] postgres@habr STATEMENT:  LOCK TABLE accounts;
Waiting for data... (interrupt to abort)
```

Посмотрим, что за OID'ы (16411 и 16410) указаны в логах:
```sql
SELECT OID, relname FROM pg_class WHERE OID = 16411;

  oid  | relname
-------+---------
 16411 | accounts 

SELECT oid AS database_id, datname AS database_name FROM pg_database WHERE oid = 16410;
 database_id | database_name
-------------+---------------
       16410 | locks

-- либо так 
SELECT pg_relation_filepath('accounts');
 pg_relation_filepath
----------------------
 base/16410/16411
```

***
<div align="center"><h2>2. Три UPDATE'а и pg_locks</h2></div>

***

### 2.1 Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.

В трех терминалах начнем новые транзакции, но перед дальнейшими действиями выполним:
```sql
SELECT txid_current(), pg_backend_pid();

(I)> 
 txid_current | pg_backend_pid
--------------+----------------
          553 |           1483
(II)>
 txid_current | pg_backend_pid
--------------+----------------
          554 |           1484     
          
(III)>
 txid_current | pg_backend_pid
--------------+----------------
          555 |           1485         
```

Проверим существующие в данный момент блокировки:
```sql
SELECT 
    pid, 
    locktype, 
    CASE locktype
        WHEN 'relation' THEN relation::regclass::text
        WHEN 'transactionid' THEN transactionid::text
        WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text END AS lockid, 
    virtualxid AS virtxid, 
    transactionid AS xid, 
    mode, 
    granted 
FROM pg_locks 
ORDER BY pid, locktype;

 pid  |    locktype   | lockid   | virtxid | xid |      mode       | granted
------+---------------+----------+---------+-----+-----------------+---------
 1483 | virtualxid    |          | 6/30    |     | ExclusiveLock   | t
 1484 | virtualxid    |          | 3/29    |     | ExclusiveLock   | t
 1485 | virtualxid    |          | 4/130   |     | ExclusiveLock   | t
```

Наблюдаем что, транзакция всегда удерживает исключительную (ExclusiveLock) блокировку собственного номера, в данном 
случае — виртуального (`virtxid`)

Далее, начиная с первого терминала выполним запрос (ожидаемо во втором и третьем терминале запросы "уснут" в ожидании блокировки):
```sql
UPDATE accounts SET amount = amount + 1000 WHERE acc_no = 1;
```

### 2.2 Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

Посмотрим на блокировки:
```sql
SELECT 
    pid, 
    locktype, 
    CASE locktype
        WHEN 'relation' THEN relation::regclass::text
        WHEN 'transactionid' THEN transactionid::text
        WHEN 'tuple' THEN relation::regclass::text||':'||tuple::text END AS lockid, 
    virtualxid AS virtxid, 
    transactionid AS xid, 
    mode, 
    granted 
FROM pg_locks 
ORDER BY pid, locktype;

 #  | pid  |   locktype    |      lockid         |         | xid |       mode       | granted
----+------+---------------+---------------------+---------+-----+------------------+---------
 1  | 1483 | relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 2  | 1483 | relation      | accounts            |         |     | RowExclusiveLock | t
 3  | 1483 | transactionid | 553                 |         | 553 | ExclusiveLock    | t
 4  | 1483 | virtualxid    |                     | 6/30    |     | ExclusiveLock    | t

 5  | 1484 | relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 6  | 1484 | relation      | accounts            |         |     | RowExclusiveLock | t
 7  | 1484 | tuple         | accounts:1          |         |     | ExclusiveLock    | t
 8  | 1484 | virtualxid    |                     | 3/29    |     | ExclusiveLock    | t
 9  | 1484 | transactionid | 554                 |         | 554 | ExclusiveLock    | t
 10 | 1484 | transactionid | 553                 |         | 553 | ShareLock        | f

 11 | 1485 | relation      | accounts_pkey       |         |     | RowExclusiveLock | t
 12 | 1485 | relation      | accounts            |         |     | RowExclusiveLock | t
 13 | 1485 | tuple         | accounts:1          |         |     | ExclusiveLock    | f
 14 | 1485 | transactionid | 555                 |         | 555 | ExclusiveLock    | t
 15 | 1485 | virtualxid    |                     | 4/130   |     | ExclusiveLock    | t
```

Разберем, что каждая из них означает:
- остались блокировки собственного виртуального номера транзакции (#4, #8, #15);
- добавились исключительные блокировки настоящего номера (`transactionid`) транзакции (который появился, как только 
  транзакция начала изменять данные, т.е. началось выполнения `UPDATE`) (#3, #9, #14);
- появились блокировки изменяемой таблицы (`accounts` и `locktype` - `relation`) (#2, #6, #12);
- появились блокировки индекса изменяемой таблицы (`accounts_pkey`) (#1, #5, #11);
- транзакция #554 из сессии #1484 обнаружила, что строка, которую она хочет изменить, заблокирована транзакцией #553 и 
  "повисла" (ShareLock) на ожидании ее номера (granted = f) (#10);
- появились блокировки захвата исключительной блокировки изменяемой версии строки (tuple) (#7 и #13)
  Замечание: транзакция #555 сразу пытается захватить блокировку версии строки и "повисает" уже на этом шаге, пропуская 
  захват блокировки номера транзакции (ShareLock), как это сделала транзакция #554;

Когда транзакция собирается изменить строку, она выполняет следующую последовательность действий:
1. Захватывает исключительную блокировку изменяемой версии строки (tuple).
2. Если xmax и информационные биты говорят о том, что строка заблокирована, то запрашивает блокировку номера транзакции 
   xmax.
3. Прописывает свой xmax и необходимые информационные биты.
4. Освобождает блокировку версии строки.

В нашем случае транзакция #553 выполнила шаги 1. и 4., так как выполнялась первой. Транзакция #554 выполнила шаг 1. 
(#9), но остановилась на шаге 2 (#10). Транзакция #555 сразу остановилась на шаге 1.

***
<div align="center"><h2>3. Взаимоблокировка Трех Транзакций</h2></div>

***

### 3.1 Воспроизведите взаимоблокировку трех транзакций. 

В трех терминалах начнем новые транзакции, но перед дальнейшими действиями выполним:
```sql
SELECT txid_current(), pg_backend_pid();

(I)> 
 txid_current | pg_backend_pid
--------------+----------------
          569 |           1057
(II)>
 txid_current | pg_backend_pid
--------------+----------------
          570 |           1213     
          
(III)>
 txid_current | pg_backend_pid
--------------+----------------
          571 |           1170         
```

Далее последовательно выполним следующие команды:

| (I) | (II) | (III) |
|:-------------------|:-------------------|:-------------------|
| UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 1; | | |
| | UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 2; | |
| | | UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 3; |
| UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 2; | | |
| | UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 3; | |
| | | UPDATE accounts</br>SET amount = amount + 1000</br>WHERE acc_no = 1; |

На последнем запросе в терминале мы увидим сообщение о возникновении взаимоблокировки (также данная транзакция будет 
оборвана самим сервером):
```shell
ERROR:  deadlock detected
DETAIL:  Process 1170 waits for ShareLock on transaction 569; blocked by process 1057.
Process 1057 waits for ShareLock on transaction 570; blocked by process 1213.
Process 1213 waits for ShareLock on transaction 571; blocked by process 1170.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,5) in relation "accounts"
```

### 3.2 Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

В логах будет сообщение аналогичное тому, что было в терминале:

```shell
2021-08-14 19:30:59.084 UTC [1170] postgres@habr ERROR:  deadlock detected
2021-08-14 19:30:59.084 UTC [1170] postgres@habr DETAIL:  
Process 1170 waits for ShareLock on transaction 569; blocked by process 1057.
Process 1057 waits for ShareLock on transaction 570; blocked by process 1213.
Process 1213 waits for ShareLock on transaction 571; blocked by process 1170.
Process 1170: update accounts set amount = amount + 1000 where acc_no = 1;
Process 1057: update accounts set amount = amount + 1000 where acc_no = 2;
Process 1213: update accounts set amount = amount + 1000 where acc_no = 3;
```

По логам мы можем построить граф циклической зависимости, который привел к взаимоблокировке:
```
Process 1170 -> transaction 569 and Process 1057 -> transaction 570 and Process 1213 -> transaction 571 and Process 1170 -> ...
```

***
<div align="center"><h2>4. Может ли UPDATE без WHERE Привести к Взаимоблокировку</h2></div>

***
### 4.1 Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
[Из статьи на Хабре:](https://habr.com/ru/company/postgrespro/blog/465263/)
> Иногда можно получить взаимоблокировку там, где, казалось бы, ее быть никак не должно. Например, удобно и привычно 
> воспринимать команды SQL как атомарные, но возьмем UPDATE — эта команда блокирует строки по мере их обновления. Это 
> происходит не одномоментно. Поэтому если одна команда будет обновлять строки в одном порядке, а другая — в другом, они 
> могут взаимозаблокироваться.

### 4.2 Попробуйте воспроизвести такую ситуацию.
Необходимо заставить Postgres выполняя два запроса `UPDATE accounts SET amount = amount + 100;` в одной транзакции 
обновить записи для `acc_no = 1`, а затем для `acc_no = 2`, а в другой транзакции сначала для `acc_no = 3`, а уже потом
для `acc_no = 2`, то есть в обратном порядке.

Воспроизвести такую ситуацию возможно двумя способами — при помощи UPDATE'а с "замедлением" и отключением 
последовательного сканирования, как в статье выше, или же, более изящно, при помощи 
[курсоров](https://postgrespro.ru/docs/postgrespro/13/plpgsql-cursors), "идущих" в разных порядках по `acc_no`,
которые позволят выбирать (для изменения т.е. используя `SELECT FOR UPDATE`) записи в нужном порядке (фактически, 
курсор с SELECT FOR UPDATE'ом будет делать то же самое, что и UPDATE, но без изменения строк, то есть, читать строки по 
одной, в любом случае, оба запроса будут брать последовательно `ExclusiveLock` для каждой строки).

В двух терминалах начнем новые транзакции и создадим два курсора:
```sql
DECLARE accounts_cursor_acc_no_asc CURSOR FOR SELECT * FROM accounts ORDER BY acc_no ASC FOR UPDATE;
DECLARE accounts_cursor_acc_no_desc CURSOR FOR SELECT * FROM accounts ORDER BY acc_no DESC FOR UPDATE;
```

Выполним последовательно следующие запросы:
```sql
(I)> FETCH accounts_cursor_acc_no_asc;
 acc_no | amount
--------+---------
      1 | 2000.00

(I)>  FETCH accounts_cursor_acc_no_asc;
 acc_no | amount
--------+---------
      2 | 2000.00
      
(II)> FETCH accounts_cursor_acc_no_desc;
 acc_no | amount
--------+---------
      3 | 3000.00

(II)> FETCH accounts_cursor_acc_no_desc; -- awaiting

(I)>  FETCH accounts_cursor_acc_no_asc;
ERROR:  deadlock detected
DETAIL:  Process 1188 waits for ShareLock on transaction 577; blocked by process 1189.
Process 1189 waits for ShareLock on transaction 576; blocked by process 1188.
HINT:  See server log for query details.
CONTEXT:  while locking tuple (0,3) in relation "accounts"
```
Что и требовалось доказать, при попытке выбрать третью запись первым курсором, которая уже "заблокирована" вторым 
курсором мы получили взаимоблокировку, та же ситуация может произойти при параллельном UPDATE'е, если порядок выборки 
записей, по каким-либо причинам, в них окажется разный. 