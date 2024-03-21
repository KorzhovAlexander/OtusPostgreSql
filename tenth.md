# Домашнее задание
## Репликация

### Цель:
- реализовать свой миникластер на 3 ВМ

### Описание/Пошаговая инструкция выполнения домашнего задания:
**Создадим 3 VM**
```
postgres1
postgres2
postgres3
```
1) На 1 ВМ(postgres1) создаем таблицы test для записи, test2 для запросов на чтение.
```
korzhov@postgres1:~$ sudo -u postgres psql
```
```sql
create database testdb1;
create table test(id integer, object_name char(30));
create table test2(id integer, object2_name char(30));

ALTER SYSTEM SET wal_level = logical;
```

```
korzhov@postgres1:~$ sudo pg_ctlcluster 15 main1 restart
korzhov@postgres1:~$ sudo -u postgres psql
postgres=# \c testdb1
You are now connected to database "testdb1" as user "postgres".
testdb1=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```
2) Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
**на всех ВМ в postgresql.conf поставил listen_address = '0.0.0.0'. В pg_hba.conf добавил строчку:**
```
host  all  all 0.0.0.0/0 md5
```

**На всех ВМ включил wal_level = 'replica'**

**На всех ВМ создал Базы:**
```
postrges1 - testdb1
postrges2 - testdb2
postrges3 - testdb3
```
**Подписка создалась, но с ошибкой, что публикации на 2 ВМ пока нет.**

```sql
CREATE PUBLICATION test_pub FOR TABLE test;
```
```
testdb1=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test"
```
```sql
create subscription test2_sub connection 'host=84.201.156.14 port=5432 user=postgres password=pass123 dbname=testdb2' PUBLICATION test2_pub WITH (copy_data = true);
```
```
WARNING:  publication "test2_pub" does not exist on the publisher
NOTICE:  created replication slot "test2_sub" on publisher
CREATE SUBSCRIPTION
```
3) На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
```sql
create table test(id integer, object_name char(30));
create table test2(id integer, object2_name char(30));
```
4) Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
**Теперь создаем публикацию и обратную подписку на 2 ВМ:**
```sql
CREATE PUBLICATION test2_pub FOR TABLE test2;
```
```
testdb2=# \dRp+
                           Publication test2_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.test2"
```
```sql
create subscription test_sub connection 'host=XXX.XXX.XX.XX port=5432 user=postgres password=pass123 dbname=te
stdb1' PUBLICATION test_pub WITH (copy_data = true);
```

**В обеих ВМ нет записей в таблицах test и test2:**
```sql
select * from test;
```
```
 id | object_name
----+-------------
(0 rows)
```

```sql
select * from test2;
```
```
 id | object2_name
----+--------------
(0 rows)
```

```sql
select * from test;
```
```
 id | object_name
----+-------------
(0 rows)
```

```sql
select * from test2;
```
```
id | object2_name
----+--------------
(0 rows)
```

**На 1 ВМ добавим запись в таблицу test, на 2 ВМ добавим запись в таблицу test2:**
```sql
insert into test values(1, 'test_obj1');
```

```sql
insert into test2 values(1, 'test_obj2');
```

**Записи появились во всех таблицах обеих ВМ:**
```sql
select * from test;
```
```
 id |          object_name
----+--------------------------------
  1 | test_obj1
(1 row)
```

```sql
select * from test2;
```
```
 id |          object2_name
----+--------------------------------
  1 | test_obj2
(1 row)
```

```sql
select * from test;
```
```
 id |          object_name
----+--------------------------------
  1 | test_obj1
(1 row)
```

```sql
select * from test2;
```
```
 id |          object2_name
----+--------------------------------
  1 | test_obj2
(1 row)
```

5) 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
```sql
create table test(id integer, object_name char(30));
create table test2(id integer, object2_name char(30));
create subscription test_sub_repl connection 'host=XXX.XXX.XX.XX port=5432 user=postgres password=pass123 dbname=testdb1' PUBLICATION test_pub WITH (copy_data = true);
create subscription test2_sub_repl connection 'host=YY.YYY.YYY.YY port=5432 user=postgres password=pass123 dbname=testdb2' PUBLICATION test2_pub WITH (copy_data = true);
```

**Данные в таблицах на 3 ВМ появились:**
```sql
testdb3=# select * from test;
 id |          object_name
----+--------------------------------
  1 | test_obj1
(1 row)

testdb3=# select * from test2;
 id |          object2_name
----+--------------------------------
  1 | test_obj2
(1 row)
```
