# Тема: Логический уровень PostgreSQL
1. Создал виртуальную машину, установил PostgreSQL 14.
2. Проверил работоспособность 
3. Зашел в psql под пользователем postgres
```
sudo -u postgres psql
```
4. Создал новую базу данных testdb
```
create database testdb
```
5. Зашел в БД testdb (\c testdb)
6. создайте новую схему testnm
```
create schema testnm
```
7. Создал новую таблицу t1 и заполнил данными:
```
create table testnm.t1(c1 integer);
insert
into
    testnm.t1
values(1);
```
8. Cоздаk новую роль readonly, дал роли readonly право на подключение к БД testdb, 
дал роли readonly право на использование схемы testnm, дал роли readonly
право на select для всех таблиц схемы testnm
```
create role readonly;

grant connect on
database testdb to readonly;

grant usage on
schema testnm to readonly;

grant
select
on
    all tables in schema testnm to readonly;
```
9. Создал пользователя testread с паролем test123, дал роль readonly пользователю testread:
```
create user testread with password 'test123';

grant readonly to testread;
```
10. Переподключился под пользователя testread к БД testdb
```
psql -U testread -h 127.0.0.1 -d testdb -W
```
11. Выполнил select таблицы t1
```
select
    *
from
    testnm.t1 
```
12. Select прошел успешно
13. Читая шпаргалку понятно, что расчет был на то, что таблица должна была создаться в схеме public, если явно не указать схему при создании.
14. Выполнил команду создания таблицы t2
```
create table testnm.t2(c1 integer);
```
Выполнить create не вышло из-за отсутсвия прав: 
```
ERROR:  permission denied for schema testnm
LINE 1: create table testnm.t2(c1 integer);
```
попробовал выполнить в схеме public:
```
create table public.t2(c1 integer);
```
Создание прошло успешно.
Добавить таблицу в схему public вышло из-за наличия дефолтных прав у роли readonly.
15. **Итог:** После просмотра шпаргалки получилось лучше понять суть домашнего задания.
