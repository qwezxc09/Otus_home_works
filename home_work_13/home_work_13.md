# Виды индексов. Работа с индексами и оптимизация запросов

1. Создал 5 таблиц (данные таблицы взяты из практики вебинара): 
```SQL
create table hwindex."SimpleIndex" (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);

create table hwindex."FullTextSearchingIndexTable" (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);

create table hwindex."FullTextSearchingIndexTable2" (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);

create table hwindex."PartialIndex" (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);

create table hwindex."MulticolumnIndexes" (
    id int,
    user_id int,
    order_date date,
    status text,
    some_text text
);
```

| Таблица                      | Для чего                                                                |
| ---------------------------- | --------------------------------------------------------------------    |
| SimpleIndex                  | Для самого простого индекса                                             |
| FullTextSearchingIndexTable  | Для индекса полнотекстового поиска с добавлением столбца типа tsvector  |
| FullTextSearchingIndexTable2 | Для индекса полнотекстового поиска  |
| PartialIndex                 | Для частичного индекс                                                   |
| MulticolumnIndexes           | Для составного индекса                                                  |

2. Добавил к таблице FullTextSearchingIndexTable новый столбец и заполнил все таблицы данными (данные также из практики вебинара):

```SQL
insert into hwindex."SimpleIndex"(id, user_id, order_date, status, some_text)
select generate_series, (random() * 70), date'2019-01-01' + (random() * 300)::int as order_date
        , (array['returned', 'completed', 'placed', 'shipped'])[(random() * 4)::int]
        , concat_ws(' ', (array['go', 'space', 'sun', 'London'])[(random() * 5)::int]
            , (array['the', 'capital', 'of', 'Great', 'Britain'])[(random() * 6)::int]
            , (array['some', 'another', 'example', 'with', 'words'])[(random() * 6)::int]
            )
from generate_series(1, 1000000);



insert into hwindex."FullTextSearchingIndexTable"(id, user_id, order_date, status, some_text)
select id, user_id, order_date, status, some_text from  hwindex."SimpleIndex"


alter table hwindex."FullTextSearchingIndexTable" add column some_text_lexeme tsvector;
update hwindex."FullTextSearchingIndexTable"
set some_text_lexeme = to_tsvector(some_text);



insert into hwindex."FullTextSearchingIndexTable2"(id, user_id, order_date, status, some_text)
select id, user_id, order_date, status, some_text from  hwindex."SimpleIndex"



insert into hwindex."PartialIndex"(id, user_id, order_date, status, some_text)
select id, user_id, order_date, status, some_text from  hwindex."SimpleIndex"


insert into hwindex."MulticolumnIndexes"(id, user_id, order_date, status, some_text)
select id, user_id, order_date, status, some_text from  hwindex."SimpleIndex"
```

### Обычный индекс
3. Запустил скрипт:
```SQL
explain analyze
select * from hwindex."SimpleIndex" where id = 10;
```
**Результат**

|id| user_id  | order_date  | status    | some_text          |
|--|----------|-------------|-----------|--------------------|
|10| 38       | 2019-04-12  | returned    | sun the |

**QUERY PLAN**                                                   
```
 Gather  (cost=1000.00..14219.43 rows=1 width=34) (actual time=0.415..121.580 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "SimpleIndex"  (cost=0.00..13219.33 rows=1 width=34) (actual time=51.781..90.599 rows=0 loops=3)
         Filter: (id = 10)
         Rows Removed by Filter: 333333
 Planning Time: 0.061 ms
 Execution Time: 121.600 ms
 ```

4. Создал индекс на таблицу, на столбец id:
```SQL
create index idx_SimpleIndex_id on hwindex."SimpleIndex"(id);
```
5. Запустил скрипт:
```SQL
explain analyze
select * from hwindex."SimpleIndex" where id = 10;
```
**Результат**

**QUERY PLAN** 
```
 Index Scan using idx_simpleindex_id on "SimpleIndex"  (cost=0.42..8.44 rows=1 width=34) (actual time=0.042..0.042 rows=1 loops=1)
   Index Cond: (id = 10)
 Planning Time: 0.330 ms
 Execution Time: 0.071 ms
```
Видно явное уменьшение времени выполнения запроса и использование индекса.

### Индекс для полнотекстового поиска
6. Запустил скрипт:
```SQL
explain analyze
select * from hwindex."FullTextSearchingIndexTable" 
where some_text_lexeme @@ to_tsquery('london');
```
**Результат (несколько строк и план запроса)**
id   | user_id | order_date |  status   |       some_text        |         some_text_lexeme          

--------|---------|------------|-----------|------------------------|-----------------------------------

   4661 |      56 | 2019-06-08 | completed | London of some         | 'london':1

    510 |      12 | 2019-10-10 |           | London of some         | 'london':1

    514 |       5 | 2019-07-16 | completed | London Britain example | 'britain':2 'exampl':3 'london':1

    515 |      69 | 2019-07-26 | returned  | London Britain example | 'britain':2 'exampl':3 'london':1

    517 |       9 | 2019-04-08 | completed | London Britain         | 'britain':2 'london':1



 **QUERY PLAN** 
 ```
 Gather  (cost=1000.00..149670.70 rows=197067 width=63) (actual time=82.753..1645.039 rows=199995 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "FullTextSearchingIndexTable"  (cost=0.00..128964.00 rows=82111 width=63) (actual time=61.297..1561.700 rows=66665 loops=3)
         Filter: (some_text_lexeme @@ to_tsquery('london'::text))
         Rows Removed by Filter: 266668
 Planning Time: 1.044 ms
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 1.825 ms, Inlining 0.000 ms, Optimization 1.749 ms, Emission 30.277 ms, Total 33.850 ms
 Execution Time: 1688.809 ms
```
7. Создал индек на таблицу, на столбец some_text_lexeme:
```SQL
CREATE INDEX search_index_ftsit ON hwindex."FullTextSearchingIndexTable"  USING GIN (some_text_lexeme);
```
8. Запустил скрипт:
```SQL
explain analyze
select * from hwindex."FullTextSearchingIndexTable" 
where some_text_lexeme @@ to_tsquery('london');
```
**Результат**

 **QUERY PLAN**                                                                       
```
 Gather  (cost=1000.00..149670.70 rows=197067 width=63) (actual time=81.154..1678.798 rows=199995 loops=1)

   Workers Planned: 2

   Workers Launched: 2

   ->  Parallel Seq Scan on "FullTextSearchingIndexTable"  (cost=0.00..128964.00 rows=82111 width=63) (actual time=57.314..1605.734 rows=66665 loops=3)

         Filter: (some_text_lexeme @@ to_tsquery('london'::text))

         Rows Removed by Filter: 266668

 Planning Time: 0.143 ms

 JIT:

   Functions: 6

   Options: Inlining false, Optimization false, Expressions true, Deforming true

   Timing: Generation 1.844 ms, Inlining 0.000 ms, Optimization 1.486 ms, Emission 22.120 ms, Total 25.451 ms

 Execution Time: 1692.679 ms
```
Индекс почему-то не применяется, судя по плану запроса, разобраться почему не вышло. Но с индексом время выполнения должно уменьшиться.

9. Далее решил проверить полнотекстовый поиск не создавая отдельного столбца для этого:

```SQL
explain analyze
select * from hwindex."FullTextSearchingIndexTable2" 
where some_text @@ to_tsquery('london');
```
 **QUERY PLAN**  
 ```
 Gather  (cost=1000.00..242723.10 rows=212121 width=34) (actual time=6.626..3983.500 rows=199995 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "FullTextSearchingIndexTable2"  (cost=0.00..220511.00 rows=88384 width=34) (actual time=9.400..3933.958 rows=66665 loops=3)
         Filter: (some_text @@ to_tsquery('london'::text))
         Rows Removed by Filter: 266668
 Planning Time: 1.295 ms
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 1.917 ms, Inlining 0.000 ms, Optimization 1.781 ms, Emission 23.803 ms, Total 27.501 ms
 Execution Time: 4013.305 ms
 ```
 10. Создал индекс на таблицу, на столбец some_text:
```SQL
CREATE INDEX search_index_ft ON hwindex."FullTextSearchingIndexTable2" (some_text);
```
11. Запустил скрипт:
```SQL
explain analyze
select *
from hwindex."FullTextSearchingIndexTable2" 
where some_text @@ to_tsquery('london');
```
 **QUERY PLAN**  
 ```
 Gather  (cost=1000.00..242723.10 rows=212121 width=34) (actual time=4.859..3818.153 rows=199995 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "FullTextSearchingIndexTable2"  (cost=0.00..220511.00 rows=88384 width=34) (actual time=5.550..3774.706 rows=66665 loops=3)
         Filter: (some_text @@ to_tsquery('london'::text))
         Rows Removed by Filter: 266668
 Planning Time: 0.346 ms
 JIT:
   Functions: 6
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 9.807 ms, Inlining 0.000 ms, Optimization 1.367 ms, Emission 12.930 ms, Total 24.104 ms
 Execution Time: 3841.162 ms
 ```

Индекс почему-то не применяется, судя по плану запроса, разобраться почему не вышло. Но в данной ситуации индекс не должен сильно повлиять на скороть выполнения.

### Частичный индекс

12. Запустил скрипт:
```SQL
explain analyze
select *
from hwindex."PartialIndex" 
where id = 100;
```
 **QUERY PLAN** 
 ```
  Gather  (cost=1000.00..14219.43 rows=1 width=34) (actual time=0.299..115.231 rows=1 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "PartialIndex"  (cost=0.00..13219.33 rows=1 width=34) (actual time=48.926..85.550 rows=0 loops=3)
         Filter: (id = 100)
         Rows Removed by Filter: 333333
 Planning Time: 0.141 ms
 Execution Time: 115.252 ms
```
13. Создал индекс на таблицу, на столбец id:
```SQL
create index idx_PartialIndex_id_100 on hwindex."PartialIndex" (id) where id > 10;
```
14. Запустил скрипт:
```SQL
explain analyze
select *
from hwindex."PartialIndex" 
where id = 100;
```
 **QUERY PLAN** 
 ```
 Index Scan using idx_partialindex_id_100 on "PartialIndex"  (cost=0.42..8.44 rows=1 width=34) (actual time=0.035..0.036 rows=1 loops=1)
   Index Cond: (id = 100)
 Planning Time: 0.297 ms
 Execution Time: 0.073 ms
```
Результат лучше.

### Составной индекс
15. Запустил скрипт:
```SQL
explain analyze
select *
from hwindex."MulticolumnIndexes" 
where user_id = 40 AND order_date > '2019-04-01';
```
 **QUERY PLAN** 
```
 Gather  (cost=1000.00..16218.80 rows=9578 width=34) (actual time=0.309..128.182 rows=9862 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on "MulticolumnIndexes"  (cost=0.00..14261.00 rows=3991 width=34) (actual time=0.048..98.666 rows=3287 loops=3)
         Filter: ((order_date > '2019-04-01'::date) AND (user_id = 40))
         Rows Removed by Filter: 330046
 Planning Time: 0.059 ms
 Execution Time: 128.829 ms
```
16. Создал индекс на таблицу, на столбец id:
```SQL
create index idx_MultiIndex on hwindex."MulticolumnIndexes"(user_id,order_date);
```
17. Запустил скрипт:
```SQL
explain analyze
select *
from hwindex."MulticolumnIndexes" 
where user_id = 40 AND order_date > '2019-04-01';
```
**QUERY PLAN** 
```
 Bitmap Heap Scan on "MulticolumnIndexes"  (cost=134.60..8699.99 rows=9578 width=34) (actual time=1.632..8.849 rows=9862 loops=1)
   Recheck Cond: ((user_id = 40) AND (order_date > '2019-04-01'::date))
   Heap Blocks: exact=5640
   ->  Bitmap Index Scan on idx_multiindex  (cost=0.00..132.21 rows=9578 width=0) (actual time=0.801..0.802 rows=9862 loops=1)
         Index Cond: ((user_id = 40) AND (order_date > '2019-04-01'::date))
 Planning Time: 0.222 ms
 Execution Time: 9.258 ms
```

Работает быстрее.