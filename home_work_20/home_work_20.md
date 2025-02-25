# Резервное копирование и восстановление
1. Создал БД, схему и таблицу, таблицу заполнил автосгенерированными данными, заранее создал вторую таблицу и заранее создал вторую БД (для задания с pg_dump):
```SQL
create database "backupDB"
create database "backupDB2"
create schema backupschema
    
create table backupschema.backuptable (
                                        id text
 )

    insert into backupschema.backuptable(id)
select
    concat('id ', generate_series)
from
    generate_series(1, 100);


create table backupschema.backuptable2(
                                          id text
)
```
2. Зашел под пользователем postgres
```
su postgres
```
3. Создал каталог buckup
```
mkdir /var/lib/postgresql/backup
```
4. Зашел в psql
```
psql
```
5. С помощью команды copy скопировал данные из backuptable в файл backuptable.sql
```
\copy backupschema.backuptable to '/var/lib/postgresql/backup/backuptable.sql';
```
6. С помощью команды copy востановил данные из файла в таблицу backuptable2
```
copy backupschema.backuptable2 from '/var/lib/postgresql/backup/backuptable.sql';
```
7. Далее необходимо было сделать бекап двух таблиц из БД. Выполнил с помощью команды ниже:
```
pg_dump -d backupDB -U postgres -Fc  -t backupschema.backuptable -t backupschema.backuptable2 > /var/lib/postgresql/backup/dump.gz
```
8. Так как при pg_dump, не может добавиться создание схемы (в ситуациях когда нужно сделать dump конкретных таблиц) во второй БД создал данную схему командой:
```SQL
create schema backupschema
```
9. Выполнил восстановление второй таблицы во вторую БД
```
pg_restore -d backupDB2 -U postgres -t backuptable2  /var/lib/postgresql/backup/dump.gz 
```
10. Таблица успешно восстановилась во второй БД