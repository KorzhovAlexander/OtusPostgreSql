# Домашнее задание
## Виды индексов. Работа с индексами и оптимизация запросов 

### Описание/Пошаговая инструкция выполнения домашнего задания:
1 вариант:
Создать индексы на БД, которые ускорят доступ к данным.
В данном задании тренируются навыки:
- определения узких мест
- написания запросов для создания индекса
- оптимизации


Необходимо:
1) Создать индекс к какой-либо из таблиц вашей БД.
2) Прислать текстом результат команды explain, в которой используется данный индекс
3) Реализовать индекс для полнотекстового поиска
4) Реализовать индекс на часть таблицы или индекс на поле с функцией
5) Создать индекс на несколько полей
6) Написать комментарии к каждому из индексов
7) Описать что и как делали и с какими проблемами столкнулись

Ход решения:

**Создал ВМ. Установил 15 Постгрес.**

**Создал БД testdb. Создал таблицу test_table с 1 миллионом сгенерированных записей.**
```sql
create table test_table as 
select 
  generate_series(1,1000000) as id,
  md5(random()::text)::char(10) as serial_number;
```
**План запроса к таблице без индексов:**
```sql
 explain
 select * from test_table where id = 1000;
```
```
QUERY PLAN                                                         |
-------------------------------------------------------------------+
Seq Scan on test_table  (cost=0.00..15206.00 rows=1000000 width=15)|
```
**1) Создадим уникальный индекс по колонке id**
```sql
create unique index test_table_ui01 on test_table(id);
```
**2) Теперь используется индекс при фильтрации по id:**
```sql
 explain
 select * from test_table where id = 1000;
```
```
QUERY PLAN                                                                       |
---------------------------------------------------------------------------------+
Index Scan using test_table_ui01 on test_table  (cost=0.42..8.44 rows=1 width=15)|
  Index Cond: (id = 1000)                                                        |
```
**3) Добавим колонку serial_number_tsvector типа tsvector для полнотекстового поиска и создадим индекс по этому полю:**
```sql
alter table test_table add column serial_number_tsvector tsvector;

update test_table set serial_number_tsvector = to_tsvector(serial_number);

create index test_table_gin_i01 ON test_table using gin (serial_number_tsvector);
```
**Индекс стал использоваться для фильтрации по полю serial_number_tsvector:**
```sql
explain
select *
from test_table
where serial_number_tsvector @@ to_tsquery('0ba39e929a');
```
```
QUERY PLAN                                                                       |
---------------------------------------------------------------------------------+
Bitmap Heap Scan on test_table  (cost=20.51..156.33 rows=33 width=38)            |
  Recheck Cond: (serial_number_tsvector @@ to_tsquery('0ba39e929a'::text))       |
  ->  Bitmap Index Scan on test_table_gin_i01  (cost=0.00..20.50 rows=33 width=0)|
        Index Cond: (serial_number_tsvector @@ to_tsquery('0ba39e929a'::text))   |
```
**4) Пересоздадим уникальный индекс, но с условием, что в него попадут только id < 100:**
```sql
drop index test_table_ui01;
create unique index test_table_ui01 on test_table(id) where id < 100;
```
**При запросе к таблице с фильтром id = 99 индекс работает, а при запросе с id = 101 - нет:**
```sql
explain
select * from test_table where id = 99;
```
```
QUERY PLAN                                                                       |
---------------------------------------------------------------------------------+
Index Scan using test_table_ui01 on test_table  (cost=0.14..8.16 rows=1 width=38)|
  Index Cond: (id = 99)
```
```sql
explain
select * from test_table where id = 101;
```
```
QUERY PLAN                                                                  |
----------------------------------------------------------------------------+
Gather  (cost=1000.00..20026.43 rows=1 width=38)                            |
  Workers Planned: 2                                                        |
  ->  Parallel Seq Scan on test_table  (cost=0.00..19026.33 rows=1 width=38)|
        Filter: (id = 101)                                                  |
```
**5) Создадим индекс на 2 поля id + serial_number:**
```sql
create index test_table_i02 on test_table(id, serial_number);
```
**Теперь при запросах по id или по id + serial_number будет подключаться индекс:**
```sql
explain
select * from test_table where id = 101;
```
```
QUERY PLAN                                                                      |
--------------------------------------------------------------------------------+
Index Scan using test_table_i02 on test_table  (cost=0.42..8.44 rows=1 width=38)|
  Index Cond: (id = 101)
```
```sql
explain
select * from test_table where id = 101 and serial_number = '1af3f060fe';
```
```
QUERY PLAN                                                                      |
--------------------------------------------------------------------------------+
Index Scan using test_table_i02 on test_table  (cost=0.42..8.45 rows=1 width=38)|
  Index Cond: ((id = 101) AND (serial_number = '1af3f060fe'::bpchar))           |
```
**Если взять фильтр только по serial_number, то индекс не подключится:**
```sql
explain
select * from test_table where serial_number = '1af3f060fe';
```
```
QUERY PLAN                                                                  |
----------------------------------------------------------------------------+
Gather  (cost=1000.00..20026.43 rows=1 width=38)                            |
  Workers Planned: 2                                                        |
  ->  Parallel Seq Scan on test_table  (cost=0.00..19026.33 rows=1 width=38)|
        Filter: (serial_number = '1af3f060fe'::bpchar)                      |
```

**Только если использовать set enable_seqscan='off', то включается индекс:**
```sql
set enable_seqscan='off';
explain
select * from test_table where serial_number = '1af3f060fe';
```
```
QUERY PLAN                                                                          |
------------------------------------------------------------------------------------+
Index Scan using test_table_i02 on test_table  (cost=0.42..22896.43 rows=1 width=38)|
  Index Cond: (serial_number = '1af3f060fe'::bpchar)                                |
```