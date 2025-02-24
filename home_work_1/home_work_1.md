# Тема: SQL и реляционные СУБД. Введение в PostgreSQL
1. Cоздана таблица "persons" и заполненена первоначальными данными:

``` 
create table persons(id serial,
                     first_name text,
                     second_name text);

insert
into
    persons(first_name,
            second_name)
values('name1',
       'sername1');

insert
into
    persons(first_name,
            second_name)
values('name2',
       'sername2');

commit;
```
2. Открыто две новые сессии и проверен текущий уровень изоляции:
```
show transaction isolation level
```
Результат:
| transaction_isolation |
| --- |
| read committed |
3. В первой сессии добавлена новая запись:
```
insert
	into
	persons(first_name,
	second_name)
values('name3',
'sername3')
```
4. Во второй сессии выполнена комманда:
```
select * from persons
```
Результат:
| id|first_name | second_name |
| --- | --- | --- |
| 1 | name1 | sername1 |
| 2 | name2 | sername2 |
Новая запись во второй сессии не отобразилась т.к. не был выполнен commit первой трнзакции. При уровне изоляции транзакции "read uncommited" присутствовала бы аномалия "грязного чтения" и запись была бы видна до коммита. 5. Выполнен commit первой транзакции. 6. Во второй сессии выполнена комманда:
```
select * from persons
```
Результат:
| id | first_name | second_name |
| --- | --- | --- |
| 1 | name1 | sername1 |
| 2 | name2 | sername2 |
| 3 | name3 | sername3 |
Новая запись во второй сессии отобразилась т.к. был выполнен commit первой трнзакции. 
7. Завершены две старые сессии. 
8. Открыто две новые ссесия и изменен текущий уровень изоляции на repeatable read:
```
set transaction isolation level repeatable read;
```
9. В первой сессии добавлена новая запись:
```
insert
into
    persons(first_name,
            second_name)
values('name4',
       'sername4');
```
10. Во второй сессии выполнена комманда:
```
select * from persons
```
Результат:
| id | first_name | second_name |
| --- | --- | --- |
| 1 | name1 | sername1 |
| 2 | name2 | sername2 |
| 3 | name3 | sername3 |
Новая запись во второй сессии не отобразилась т.к. не был выполнено commit, а уровень изоляции repeatable read более строгий чем read commited. 
11. Выполнен commit первой транзакции. 
12. Во второй сессии выполнена комманда:
```
select * from persons
```
Результат:
| id | first_name | second_name |
| --- | --- | --- |
| 1 | name1 | sername1 |
| 2 | name2 | sername2 |
| 3 | name3 | sername3 |
Новая запись во второй сессии не отобразилась т.к. уровень изоляции repeatable read более строгий чем read commited и не допускает аномалии неповторяющегося чтения. В новых сессиях, которые будут созданы после успешного коммита второй транзакции, новая запись будет отображатся 
13. Закрты обе сессии