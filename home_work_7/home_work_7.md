# Тема: MVCC, vacuum и autovacuum
### Основное задание
1. Создал виртуальную машину, установил PostgreSQL 15
2. Проверил работоспособность 
3. Инициализировал pgbench на БД postgres коммандой:
```
pgbench -h 127.0.0.1 -p 5432 -U postgres -i postgres
```
4. Запустил pgbench на БД postgres коммандой:
```
pgbench -h 127.0.0.1 -p 5432 -c8 -P 6 -T 60 -U postgres postgres
```

Вывод с дефолтными параметрами показал следующие результаты:

```
progress: 6.0 s, 4006.5 tps, lat 1.972 ms stddev 0.995, 0 failed
progress: 12.0 s, 3576.5 tps, lat 2.236 ms stddev 1.372, 0 failed
progress: 18.0 s, 3386.8 tps, lat 2.361 ms stddev 1.561, 0 failed
progress: 24.0 s, 3466.3 tps, lat 2.307 ms stddev 1.536, 0 failed
progress: 30.0 s, 3602.2 tps, lat 2.220 ms stddev 1.375, 0 failed
progress: 36.0 s, 3506.0 tps, lat 2.280 ms stddev 1.492, 0 failed
progress: 42.0 s, 3550.2 tps, lat 2.253 ms stddev 1.429, 0 failed
progress: 48.0 s, 3470.0 tps, lat 2.304 ms stddev 1.538, 0 failed
progress: 54.0 s, 3346.8 tps, lat 2.389 ms stddev 1.515, 0 failed
progress: 60.0 s, 3504.7 tps, lat 2.282 ms stddev 1.517, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 212504
number of failed transactions: 0 (0.000%)
latency average = 2.255 ms
latency stddev = 1.440 ms
initial connection time = 70.178 ms
tps = 3545.335631 (without initial connection time)
```
### Описание значений в отчете
| Параметр                                  | Значение                                                        |
|-------------------------------------------|-----------------------------------------------------------------|
| s (second)                                | Время продолжительности теста                                   |
| tps (Transactions per Second)             | Скорость обработки транзакций в секунду                         |
| lat (latency)                             | Средняя задержка для одной транзакции составляет в милисекундах |
| stddev                                    | Колебания задержки транзакций                                   |
| Конечное значение (0 failed)              | Количество неудачных транзакций                                 |
| number of transactions actually processed | Общее количество обработанных транзакций                        |
| number of failed transactions             | Количество неудачных транзакций                                 |

5. Выставил настройки PostgreSQl в файле postgressql.conf на настройки из материала к ДЗ и рестартанул кластер. Запустил pgbench повторно
Вывод представлен ниже: 
```
progress: 6.0 s, 3533.3 tps, lat 2.241 ms stddev 1.461, 0 failed
progress: 12.0 s, 3561.2 tps, lat 2.246 ms stddev 1.346, 0 failed
progress: 18.0 s, 3575.0 tps, lat 2.236 ms stddev 1.391, 0 failed
progress: 24.0 s, 3623.7 tps, lat 2.207 ms stddev 1.395, 0 failed
progress: 30.0 s, 3576.3 tps, lat 2.236 ms stddev 1.333, 0 failed
progress: 36.0 s, 4137.5 tps, lat 1.933 ms stddev 1.033, 0 failed
progress: 42.0 s, 3594.8 tps, lat 2.224 ms stddev 1.440, 0 failed
progress: 48.0 s, 3565.3 tps, lat 2.243 ms stddev 1.378, 0 failed
progress: 54.0 s, 3613.0 tps, lat 2.213 ms stddev 1.331, 0 failed
progress: 60.0 s, 3590.0 tps, lat 2.227 ms stddev 1.460, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 218229
number of failed transactions: 0 (0.000%)
latency average = 2.197 ms
latency stddev = 1.361 ms
initial connection time = 58.571 ms
tps = 3640.155009 (without initial connection time)
```
Координальных отличий замечено не было
6. Создал таблицу с текстовым полем (id), заполнил сгенерированными данным, проверил наличие данных и посмотрел текущий размер таблицы:
```SQL
create table test(c1 text);
create table public.testTable(
                                 id text
)

    insert
	into
	public.testTable(id)
select
    concat('id ',
           generate_series)
from
    generate_series(1,
                    1000000);
select
    *
from
    public.testTable
```
Итоговый вывод представлен ниже (первые 5 строк):

|id|
|--|
|id 1|
|id 2|
|id 3|
|id 4|
|id 5|**

|pg_size_pretty|
|--------------|
|47 MB|

7. 5 раз обновил строки: 
```SQL
call update_data(5)
```

8. Посмотрел количество мертвых строк в таблице и время последнего автовакуума:
```SQL
select
    c.relname,
    s.n_dead_tup,
    s.last_autovacuum
from
    pg_stat_user_tables s
        join pg_class c on
        s.relname = c.relname
where
    s.n_dead_tup > (current_setting('autovacuum_vacuum_threshold')::int
+ (current_setting('autovacuum_vacuum_scale_factor')::float * c.reltuples));
```
Результат: 

|relname|n_dead_tup|last_autovacuum|
|-------|----------|---------------|
|testtable|5000000|2024-10-27 19:23:46.708 +0700|

9. Дождался выполнения автовакуума. 5 раз обновил строки:
```SQL
call update_data(5)
```
10. Посмотрел текущий размер таблицы:
```SQL
select
*
from
public.testTable
```
**Результат:**

| pg_size_pretty |
|----------------|
| 375 MB         |

11. Отключил автовакуум на таблице testTable: 
```SQL
select
*
from
public.testTable
```
Результат:

| pg_size_pretty |
|----------------|
| 857 MB         |

Большой объем таблицы связан с отключением autovacuum. При включении данной функции размер таблицы будет уменьшен так как будут удалены 10 млн мертвых строк. 
Так же для уменьшения объема таблицы можно сделать vacuum full. 
