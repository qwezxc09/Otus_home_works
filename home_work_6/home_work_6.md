# Тема: Настройка PostgreSQL

 ### Информация о системе

| Параметр                         | Значение |
|----------------------------------|----------|
| Количество ядер                  | 2        |
| Значение оперативной памяти (ГБ) | 2        |
| Память жесткого диска (ГБ)       | 64       |
| Версия PostgreSQL                | 15       |

1. Создал БД pgbench
2. Запустил команду pgbench на дефолтных настройках:
```
pgbench -h 127.0.0.1 -p 5432 -U postgres -c 50 -j 2 -P 60 -T 600  pgbench
```

Вывод с дефолтными параметрами показал следующие результаты:

```
progress: 60.0 s, 3381.2 tps, lat 14.728 ms stddev 13.573, 0 failed
progress: 120.0 s, 3566.6 tps, lat 14.018 ms stddev 9.051, 0 failed
progress: 180.0 s, 3673.7 tps, lat 13.611 ms stddev 5.887, 0 failed
progress: 240.0 s, 3673.5 tps, lat 13.610 ms stddev 5.467, 0 failed
progress: 300.0 s, 3717.7 tps, lat 13.448 ms stddev 5.818, 0 failed
progress: 360.0 s, 3687.1 tps, lat 13.559 ms stddev 8.284, 0 failed
progress: 420.0 s, 3711.5 tps, lat 13.471 ms stddev 6.081, 0 failed
progress: 480.0 s, 3580.2 tps, lat 13.965 ms stddev 5.972, 0 failed
progress: 540.0 s, 3708.5 tps, lat 13.482 ms stddev 5.776, 0 failed
progress: 600.0 s, 3646.6 tps, lat 13.710 ms stddev 5.799, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2180845
number of failed transactions: 0 (0.000%)
latency average = 13.750 ms
latency stddev = 7.505 ms
initial connection time = 225.903 ms
tps = 3635.934065 (without initial connection time)
```

4. После зашел на сайт https://pgtune.leopard.in.ua для получения рекомендуемых значений 
### Рекомендуемые значения
```SQL
max_connections = 50
shared_buffers = 128MB
effective_cache_size = 512MB
maintenance_work_mem = 128MB
checkpoint_completion_target = 0.9
wal_buffers = 3932kB
default_statistics_target = 100
random_page_cost = 4
effective_io_concurrency = 2
work_mem = 1092kB
huge_pages = off
min_wal_size = 100MB
max_wal_size = 2GB
wal_level = minimal
max_wal_senders = 0
```
### Результат после установки рекомендуемых значений
```CMD
progress: 60.0 s, 3544.7 tps, lat 14.057 ms stddev 7.466, 0 failed
progress: 120.0 s, 3611.8 tps, lat 13.841 ms stddev 7.699, 0 failed
progress: 180.0 s, 3568.9 tps, lat 14.011 ms stddev 7.125, 0 failed
progress: 240.0 s, 3559.2 tps, lat 14.047 ms stddev 7.012, 0 failed
progress: 300.0 s, 3593.8 tps, lat 13.912 ms stddev 7.024, 0 failed
progress: 360.0 s, 3662.3 tps, lat 13.652 ms stddev 6.978, 0 failed
progress: 420.0 s, 3642.5 tps, lat 13.726 ms stddev 7.474, 0 failed
progress: 480.0 s, 3548.4 tps, lat 14.090 ms stddev 7.163, 0 failed
progress: 540.0 s, 3599.2 tps, lat 13.892 ms stddev 6.706, 0 failed
progress: 600.0 s, 3622.1 tps, lat 13.803 ms stddev 15.401, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2157230
number of failed transactions: 0 (0.000%)
latency average = 13.902 ms
latency stddev = 8.391 ms
initial connection time = 196.486 ms
tps = 3595.940812 (without initial connection time)
```

**5. Далее управлял параметрами указанными ниже:**

| Параметр             | Описание параметра                                                        |
|----------------------|---------------------------------------------------------------------------|
| autovacuum           | Запускает подпроцесс автоочистки.                                         |
| checkpoint_timeout   | Задаёт максимальное время между автоматическими контрольными точками WAL. |
| effective_cache_size | Подсказывает планировщику примерный общий размер кешей данных.            |
| maintenance_work_mem | Задаёт предельный объём памяти для операций по обслуживанию.              |
| max_connections      | Задаёт максимально возможное число подключений.                           |
| max_wal_size         | Задаёт размер WAL, при котором инициируется контрольная точка.            |
| shared_buffers       | Задаёт количество буферов в разделяемой памяти, используемых сервером.    |
| synchronous_commit   | Задаёт уровень синхронизации текущей транзакции.                          |
| temp_buffers         | Задаёт предельное число временных буферов на один сеанс.                  |
| wal_buffers          | Задаёт число буферов дисковых страниц в разделяемой памяти для WAL.       |
| wal_level            | Задаёт уровень информации, записываемой в WAL.                            |
| work_mem             | Задаёт предельный объём памяти для рабочих пространств запросов.          |

**По рекомендациям из источников ниже:**

| Параметр             | Описание параметра                                                 |
|----------------------|--------------------------------------------------------------------|
| autovacuum           | on                                                                 |
| checkpoint_timeout   | 10min                                                              |
| effective_cache_size | Рекомендуется выделить 50-75% от общей оперативной памяти (1536МБ) |
| maintenance_work_mem | 128MB                                                              |
| max_connections      | 50                                                                 |
| max_wal_size         | 1GB                                                                |
| shared_buffers       | Рекомендуется выделить 20-25% от общей оперативной памяти (512МБ)  |
| synchronous_commit   | off                                                                |
| temp_buffers         | 16MB                                                               |
| wal_buffers          | 4MB                                                                |
| wal_level            | minimal                                                            |
| work_mem             | 4MB                                                                |

### Результат после установки рекомендуемых значений из источника
```CMD
progress: 60.0 s, 3582.7 tps, lat 13.908 ms stddev 12.687, 0 failed
progress: 120.0 s, 3651.1 tps, lat 13.691 ms stddev 6.157, 0 failed
progress: 180.0 s, 3651.1 tps, lat 13.695 ms stddev 6.297, 0 failed
progress: 240.0 s, 3615.8 tps, lat 13.828 ms stddev 6.269, 0 failed
progress: 300.0 s, 3705.4 tps, lat 13.491 ms stddev 7.505, 0 failed
progress: 360.0 s, 3685.4 tps, lat 13.568 ms stddev 6.938, 0 failed
progress: 420.0 s, 3558.2 tps, lat 14.051 ms stddev 8.576, 0 failed
progress: 480.0 s, 3693.2 tps, lat 13.538 ms stddev 7.066, 0 failed
progress: 540.0 s, 3702.1 tps, lat 13.505 ms stddev 6.030, 0 failed
progress: 600.0 s, 3590.1 tps, lat 13.927 ms stddev 7.007, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 150
query mode: simple
number of clients: 50
number of threads: 2
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 2186166
number of failed transactions: 0 (0.000%)
latency average = 13.718 ms
latency stddev = 7.676 ms
initial connection time = 195.557 ms
tps = 3644.465520 (without initial connection time)
```

**Итог:** После установки рекомендуемых значений менял 
по разному параметры, но особых результатов это не дало. 