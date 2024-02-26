# Домашнее задание

## Настройка autovacuum с учетом особеностей производительности

### Цель:

- запустить нагрузочный тест pgbench
- настроить параметры autovacuum
- проверить работу autovacuum

### Описание/Пошаговая инструкция выполнения домашнего задания:

1) Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
2) Установить на него PostgreSQL 15 с дефолтными настройками

```
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt -y install postgresql-15
```

3) Создать БД для тестов: выполнить pgbench -i postgres
   **Под обычным пользователем не хватает прав, нужно стать пользователем postgres**

```
korzhov@postgres:~$ pgbench -i postgres
pgbench: error: connection to server on socket "/var/run/postgresql/.s.PGSQL.5432" failed: FATAL:  role "korzhov" does not exist
pgbench: error: could not create connection for initialization
korzhov@postgres:~$ sudo su postgres
postgres@postgres:/home/korzhov$
postgres@postgres:/home/korzhov$
postgres@postgres:/home/korzhov$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.07 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 2.14 s (drop tables 0.00 s, create tables 0.00 s, client-side generate 0.68 s, vacuum 0.08 s, primary keys 1.37 s).
```

4) Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres

```
postgres@postgres:/home/korzhov$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 691.7 tps, lat 11.528 ms stddev 10.036, 0 failed
progress: 12.0 s, 727.3 tps, lat 10.997 ms stddev 6.754, 0 failed
progress: 18.0 s, 679.8 tps, lat 11.753 ms stddev 7.210, 0 failed
progress: 24.0 s, 699.5 tps, lat 11.451 ms stddev 7.441, 0 failed
progress: 30.0 s, 481.7 tps, lat 16.550 ms stddev 13.642, 0 failed
progress: 36.0 s, 615.5 tps, lat 13.039 ms stddev 11.438, 0 failed
progress: 42.0 s, 729.5 tps, lat 10.970 ms stddev 7.343, 0 failed
progress: 48.0 s, 621.7 tps, lat 12.870 ms stddev 50.842, 0 failed
progress: 54.0 s, 767.3 tps, lat 10.424 ms stddev 6.804, 0 failed
progress: 60.0 s, 493.2 tps, lat 16.189 ms stddev 15.007, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 39051
number of failed transactions: 0 (0.000%)
latency average = 12.291 ms
latency stddev = 18.233 ms
initial connection time = 14.822 ms
tps = 650.597861 (without initial connection time)
```

5) Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла

```
korzhov@postgres:~$ sudo pg_ctlcluster 15 main stop
korzhov@postgres:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
korzhov@postgres:~$ sudo pg_ctlcluster 15 main start
korzhov@postgres:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```

6) Протестировать заново

```
postgres@postgres:/etc/postgresql/15/main$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.2 (Ubuntu 15.2-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 527.3 tps, lat 15.112 ms stddev 54.948, 0 failed
progress: 12.0 s, 509.8 tps, lat 15.692 ms stddev 14.354, 0 failed
progress: 18.0 s, 772.8 tps, lat 10.353 ms stddev 6.523, 0 failed
progress: 24.0 s, 609.5 tps, lat 13.116 ms stddev 50.817, 0 failed
progress: 30.0 s, 709.5 tps, lat 11.285 ms stddev 7.460, 0 failed
progress: 36.0 s, 765.5 tps, lat 10.421 ms stddev 6.535, 0 failed
progress: 42.0 s, 529.0 tps, lat 15.156 ms stddev 16.161, 0 failed
progress: 48.0 s, 765.5 tps, lat 10.457 ms stddev 6.174, 0 failed
progress: 54.0 s, 741.2 tps, lat 10.781 ms stddev 7.195, 0 failed
progress: 60.0 s, 618.2 tps, lat 12.900 ms stddev 8.785, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 39298
number of failed transactions: 0 (0.000%)
latency average = 12.213 ms
latency stddev = 23.628 ms
initial connection time = 15.259 ms
tps = 654.871001 (without initial connection time)
```

7) Что изменилось и почему?
   **Существенных изменений не наблюдаю. На 200+ транзакций обработалось больше_
8) Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данным в размере 1млн строк

```sql
create table test(text_field char(100));
```
```sql
INSERT INTO test(text_field) SELECT 'test_val' FROM generate_series(1,1000000);
```

9) Посмотреть размер файла с таблицей

```sql
select pg_size_pretty( pg_total_relation_size('test'));
```
```
 pg_size_pretty
----------------
 128 MB
(1 row)
```

10) 5 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
postgres=# update test set text_field = text_field || 'a';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'b';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'c';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'd';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'e';
UPDATE 1000000
```

11) Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум

```sql
SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';
```
```
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |          0 |      0 | 2024-02-25 20:43:01.127411+00
(1 row)
```

12) Подождать некоторое время, проверяя, пришел ли автовакуум
13) 5 раз обновить все строчки и добавить к каждой строчке любой символ

```sql
postgres=# update test set text_field = text_field || 'a';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'b'
UPDATE 1000000
postgres=# update test set text_field = text_field || 'c';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'd';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'e';
UPDATE 1000000
```

14) Посмотреть размер файла с таблицей

```sql
select pg_size_pretty( pg_total_relation_size('test'));
```
```
 pg_size_pretty
----------------
 384 MB
(1 row)
```

15) Отключить Автовакуум на конкретной таблице

```sql
ALTER TABLE test SET (autovacuum_enabled = off);
```

16) 10 раз обновить все строчки и добавить к каждой строчке любой символ
```sql
postgres=# update test set text_field = text_field || 'a';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'b';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'c';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'd';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'e';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'f';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'g';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'h';
UPDATE 1000000
postgres=# update test set text_field = text_field || 'i';
UPDATE 1000000
postgres=# update test set text_field = text_field || 't';
UPDATE 1000000
```

17) Посмотреть размер файла с таблицей

```sql
select pg_size_pretty( pg_total_relation_size('test'));
```
```
 pg_size_pretty
----------------
 1408 MB
(1 row)
```

18) Объясните полученный результат

```
1408 MB - 384 MB = 1024 MB
```
На каждый апдейт всей таблицы получаем увеличение места на 128 МБ, на изначальный размер таблицы.

Из 384 МБ реально было занято 128, остальные 256 были пустые из-за работы автовакуума и переиспользовались при вставке
новых записей во время апдейта.
1024 МБ нового места заняла таблица при вставках во время апдейтов.
(1024 + 256) / 10 апдейтов = 128 МБ места под каждый апдейт.



19) Не забудьте включить автовакуум)

```sql
ALTER TABLE test SET (autovacuum_enabled = on);
```