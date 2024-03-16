# Домашнее задание
## Механизм блокировок

### Цель:
- понимать как работает механизм блокировок объектов и строк

### Описание/Пошаговая инструкция выполнения домашнего задания:
1) Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.
Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

**Настройка:**
```sql
ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM SET deadlock_timeout = '0.2s';
SELECT pg_reload_conf();
```

```
 pg_reload_conf
----------------
 t
(1 row)
```

```SHOW deadlock_timeout;```
```
 deadlock_timeout
------------------
 200ms
(1 row)
```

```SHOW log_lock_waits;```
```
 log_lock_waits
----------------
 on
(1 row)
```

```sql
postgres=# create table test(number_field int);
CREATE TABLE
postgres=# insert into test(number_field) values (1);
INSERT 0 1
postgres=# insert into test(number_field) values (2);
INSERT 0 1
postgres=# insert into test(number_field) values (3);
INSERT 0 1
postgres=# select * from test;
 number_field
--------------
            1
            2
            3
(3 rows)
```
**Воспроизведение:**
**Заблокируем строчку с number_field = 1 в одной сессии:**
```sql
postgres=# begin;
BEGIN
postgres=*# update test set number_field = 0 where number_field = 1;
UPDATE 1
```

**Во второй сессии попробуем проапдейтить эту же строчку:**
```sql
postgres=# update test set number_field = 2 where number_field = 1;
```

**Подождём немного времени и скажем commit в первой сессии.**
**Посмотрим журнал, увидим в нём информацию о том, что через 200 милисекунд появилась информация о блокировке:**
```
2024-03-12 14:26:09.141 UTC [6086] postgres@postgres LOG:  process 6086 still waiting for ShareLock on transaction 737 after 200.087 ms
2024-03-12 14:26:09.141 UTC [6086] postgres@postgres DETAIL:  Process holding the lock: 5980. Wait queue: 6086.
2024-03-12 14:26:09.141 UTC [6086] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2024-03-12 14:26:09.141 UTC [6086] postgres@postgres STATEMENT:  update test set number_field = 2 where number_field = 1;
2024-03-12 14:26:22.126 UTC [6086] postgres@postgres LOG:  process 6086 acquired ShareLock on transaction 737 after 13184.926 ms
2024-03-12 14:26:22.126 UTC [6086] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2024-03-12 14:26:22.126 UTC [6086] postgres@postgres STATEMENT:  update test set number_field = 2 where number_field = 1;
```
2) Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.
**1 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          750 |           6704
(1 row)

postgres=*# update test set number_field = 3 where number_field = 2;
UPDATE 1
```

**2 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          751 |           6823
(1 row)

postgres=*# update test set number_field = 33 where number_field = 2;
```

**3 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          752 |           6889
(1 row)

postgres=*# update test set number_field = 333 where number_field = 2;
```
**Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны
Пришлите список блокировок и объясните, что значит каждая.**

**Список блокировок:**
```
   locktype    | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction | pid  |       mode       | granted | fastpath |           waitstart
---------------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+------+------------------+---------+----------+-------------------------------
 relation      |        5 |    12073 |      |       |            |               |         |       |          | 3/118              | 6704 | AccessShareLock  | t       | t        |
 relation      |        5 |    16390 |      |       |            |               |         |       |          | 3/118              | 6704 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 3/118      |               |         |       |          | 3/118              | 6704 | ExclusiveLock    | t       | t        |
 relation      |        5 |    16390 |      |       |            |               |         |       |          | 5/22               | 6889 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 5/22       |               |         |       |          | 5/22               | 6889 | ExclusiveLock    | t       | t        |
 relation      |        5 |    16390 |      |       |            |               |         |       |          | 4/76               | 6823 | RowExclusiveLock | t       | t        |
 virtualxid    |          |          |      |       | 4/76       |               |         |       |          | 4/76               | 6823 | ExclusiveLock    | t       | t        |
 tuple         |        5 |    16390 |    0 |     2 |            |               |         |       |          | 5/22               | 6889 | ExclusiveLock    | f       | f        | 2024-03-12 15:00:13.8537+00
 transactionid |          |          |      |       |            |           752 |         |       |          | 5/22               | 6889 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           750 |         |       |          | 3/118              | 6704 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           751 |         |       |          | 4/76               | 6823 | ExclusiveLock    | t       | f        |
 tuple         |        5 |    16390 |    0 |     2 |            |               |         |       |          | 4/76               | 6823 | ExclusiveLock    | t       | f        |
 transactionid |          |          |      |       |            |           750 |         |       |          | 4/76               | 6823 | ShareLock        | f       | f        | 2024-03-12 14:59:50.504259+00
(13 rows)
 ```
 - Блокировка relation = 12073 не интересует, это pg_locks.
 - 3 RowExclusiveLock блокировки строчки у таблицы test(16390) - это 3 сессии, пытающиеся сделать update строчки.
 - 3 virtualxid - виртаульные ID транзакции.
 - Блокировка transactionid для pid = 6704 - это транзакция 1 сессии, захватившая строчку
 - 2 Блокировки transactionid для pid = 6823 - это транзакция 2 сессии, пытающаяся захватившая строчку, и ShareLock ожадание 1 сессии со ссылкой на транзакцию 1 сессии (750)
 - Блокировка tuple для pid = 6823 - это ссылка на объект обновления(строчку таблицы test)
 - Блокировка transactionid для pid = 6889 - это транзакция 3 сессии, пытающаяся захватившая строчку
 - Блокировка tuple для pid = 6889 - это ссылка на объект обновления(строчку таблицы test)


3) Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

**Заблокируем по 1 строчке в каждой сессии**

**1 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          760 |           7323
(1 row)

postgres=*# update test set number_field = 2 where number_field = 1;
UPDATE 1
```

**2 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          761 |           7496
(1 row)

postgres=*# update test set number_field = 3 where number_field = 2;
UPDATE 1
```

**3 сессия**
```sql
postgres=# begin;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          762 |           7562
(1 row)

postgres=*# update test set number_field = 4 where number_field = 3;
UPDATE 1
```

**Теперь каждая сессия попробует проапдейтить строчку соседней: 1 сессия проапдейтит строчку 2 сессии, 2 сессия проапдейтит строчку 3 сессии, 3 сессия проапдейтит строчку 1 сессии**


**1 сессия**
```sql
postgres=*# update test set number_field = 3 where number_field = 2;
```
**2 сессия**
```sql
postgres=*# update test set number_field = 4 where number_field = 3;
```
**3 сессия**
```sql
postgres=*# update test set number_field = 2 where number_field = 1;
ERROR:  deadlock detected
DETAIL:  Process 7562 waits for ShareLock on transaction 760; blocked by process 7323.
Process 7323 waits for ShareLock on transaction 761; blocked by process 7496.
Process 7496 waits for ShareLock on transaction 762; blocked by process 7562.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,1) in relation "test"
```

**Как видим 3 сессия упала при попытке зацикливания, остальные остались**
```
2024-03-12 15:33:40.233 UTC [7323] postgres@postgres LOG:  process 7323 still waiting for ShareLock on transaction 761 after 200.132 ms
2024-03-12 15:33:40.233 UTC [7323] postgres@postgres DETAIL:  Process holding the lock: 7496. Wait queue: 7323.
2024-03-12 15:33:40.233 UTC [7323] postgres@postgres CONTEXT:  while updating tuple (0,2) in relation "test"
2024-03-12 15:33:40.233 UTC [7323] postgres@postgres STATEMENT:  update test set number_field = 3 where number_field = 2;
2024-03-12 15:33:49.782 UTC [7496] postgres@postgres LOG:  process 7496 still waiting for ShareLock on transaction 762 after 200.124 ms
2024-03-12 15:33:49.782 UTC [7496] postgres@postgres DETAIL:  Process holding the lock: 7562. Wait queue: 7496.
2024-03-12 15:33:49.782 UTC [7496] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "test"
2024-03-12 15:33:49.782 UTC [7496] postgres@postgres STATEMENT:  update test set number_field = 4 where number_field = 3;
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres LOG:  process 7562 detected deadlock while waiting for ShareLock on transaction 760 after 200.148 ms
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres DETAIL:  Process holding the lock: 7323. Wait queue: .
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres STATEMENT:  update test set number_field = 2 where number_field = 1;
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres ERROR:  deadlock detected
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres DETAIL:  Process 7562 waits for ShareLock on transaction 760; blocked by process 7323.
        Process 7323 waits for ShareLock on transaction 761; blocked by process 7496.
        Process 7496 waits for ShareLock on transaction 762; blocked by process 7562.
        Process 7562: update test set number_field = 2 where number_field = 1;
        Process 7323: update test set number_field = 3 where number_field = 2;
        Process 7496: update test set number_field = 4 where number_field = 3;
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres HINT:  See server log for query details.
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "test"
2024-03-12 15:33:56.335 UTC [7562] postgres@postgres STATEMENT:  update test set number_field = 2 where number_field = 1;
2024-03-12 15:33:56.336 UTC [7496] postgres@postgres LOG:  process 7496 acquired ShareLock on transaction 762 after 6753.560 ms
2024-03-12 15:33:56.336 UTC [7496] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "test"
2024-03-12 15:33:56.336 UTC [7496] postgres@postgres STATEMENT:  update test set number_field = 4 where number_field = 3;
```

**В журнале видим блокировку строчек сессиями и в конечном итоге deadlock с описанием кто кого заблокировал и как он случился.**

4) Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
Задание со звездочкой*
Попробуйте воспроизвести такую ситуацию.

**Создадим таблицу с 1 миллионом записей**
```sql
postgres=# create table test(number_field int);
CREATE TABLE
postgres=# INSERT INTO test(number_field) SELECT floor(random() * 10)::int FROM generate_series(1,1000000);
INSERT 0 1000000
```

**1 сессия**
```sql
postgres=*# update test set number_field = number_field + 1;
```
**2 сессия**
```sql
postgres=*# update test set number_field = number_field + 11;
```
**В логе тоже видим такую картину, вторая сессия блокировалась на определенной изменённой записи 1 сессией**
```
2024-03-12 16:00:23.103 UTC [8008] postgres@postgres LOG:  process 8008 still waiting for ShareLock on transaction 773 after 200.089 ms
2024-03-12 16:00:23.103 UTC [8008] postgres@postgres DETAIL:  Process holding the lock: 7726. Wait queue: 8008.
2024-03-12 16:00:23.103 UTC [8008] postgres@postgres CONTEXT:  while updating tuple (2592,1) in relation "test"
2024-03-12 16:00:23.103 UTC [8008] postgres@postgres STATEMENT:  update test set number_field = number_field + 11;
```

**Запускал 3 раза.**
**У меня не получилось воспроизвести ситуацию, чтобы 2 сессии заблокировали друг друга. Но я подозреваю, что это возможно, если сессии начнут читать таблицу с разных страниц.**