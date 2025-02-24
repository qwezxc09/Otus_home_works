# Тема: Установка PostgreSQL
1. В Virtual Box была развернута виртуалка и установлен Docker.
2. Чтобы убедится в том, что контейнуры создаются успешно была выполнена команда:
```
sudo docker run --name some-postgres -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```
3. Были проведены шаги по созданию данных в Postgres, но после удаления контейнера данные не сохранились.
4. Корректно смотнтировать дирректорию и выполнить шаги получилоь следующей командой:
```
sudo docker run -v ~/var/lib/postgresql/data:/var/lib/postgresql/data --name PGDOCKER -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```
5. Через psql подключился к серверу и запустил скрипт создания таблицы и скрипт заполнения таблиццы данными:
```
create table public.testtable (
                                  id serial not null,
                                  value varchar not null
);

insert
into
    public.testtable
    (value)
values('test1');

insert
into
    public.testtable
    (value)
values('test2');
```
6. Удалил контейнер с сервером docker коммандами:
```
sudo docker stop PGDOCKER
sudo docker rm PGDOCKER  
```
7. Создал контейнер заново коммандой:
```
sudo docker run -v ~/var/lib/postgresql/data:/var/lib/postgresql/data --name PGDOCKER -e POSTGRES_PASSWORD=mysecretpassword -d postgres
```
8. Проверил наличие данных. Данные не пропали.