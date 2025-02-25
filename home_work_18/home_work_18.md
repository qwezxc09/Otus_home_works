# Хранимые процедуры

1. Создал схему и три таблицы: товары, продажи и витрина. Для создания воспользовался скриптом из материалов к ДЗ. Код создания ниже
```SQL
 --Создание схемы
create schema pract_functions;

--Создание таблицы товаров
create table pract_functions.goods
(
    goods_id integer primary key,
    goods_name varchar(63) not null,
    goods_price numeric(12,2) not null
        check (goods_price > 0.0)
);

--Создание таблицы продаж
create table pract_functions.sales
(
    sales_id integer generated always as identity primary key,
    goods_id integer references goods (goods_id),
    sales_time timestamp with time zone default now(),
    sales_qty integer check (sales_qty > 0)
);

--Создание таблицы Витрины
create table pract_functions.goods_sum_mart
(
    goods_name varchar(63) not null,
    sum_sale numeric(16, 2)not null
);
```
2. Создал тригер + тригерную функцию на таблицу sales, 
Код создания тригера и тригерной функции ниже:
```SQL
 -- Создание триггерной функции для таблицы sales
create or replace function change_goods_sum_mart()
returns trigger
AS
$$
declare
name_goods_sum_mart_changed varchar(63);
	difference_qte integer;
	price_goods_sum_mart_changed numeric(12, 2);
begin
	name_goods_sum_mart_changed = (select goods_name from goods
		where goods_id = coalesce(new.goods_id, old.goods_id)
								  );

	price_goods_sum_mart_changed = (select goods_price from goods
		where goods_id = coalesce(new.goods_id, old.goods_id)
								  );
	
    difference_qte = coalesce(new.sales_qty,0) - coalesce(old.sales_qty,0);
    update goods_sum_mart
    set sum_sale = sum_sale + (difference_qte * price_goods_sum_mart_changed)
    where goods_name = name_goods_sum_mart_changed;
    return null;
end;
$$
language plpgsql


-- Создание триггера для таблицы sales
create trigger tr_change_goods_sum_mart
after insert or update or delete
on pract_functions.sales
for each row
execute procedure change_goods_sum_mart();
```
3. Не совсем понял как должны обновлятся товары в ветрине (не руками же) поэтому создал триггер и для таблицы товаров:
```SQL
 -- Создание триггерной функции для таблицы sales
create or replace function new_goods_in_goods_sum_mart()
returns trigger
AS
$$
declare
begin
if new.goods_name is not null and old.goods_name is not null then
update goods_sum_mart
set goods_name = new.goods_name
where goods_name = old.goods_name;
elseif new.goods_name is not null and old.goods_name is null then 
 	insert into goods_sum_mart (goods_name, sum_sale) values 
		(new.goods_name, 0);
elseif new.goods_name is null and old.goods_name is not null then
delete from goods_sum_mart
where goods_name = old.goods_name;
end if;
return null;
end;
$$
language plpgsql


--Создание триггера для таблицы goods
create trigger tr_new_goods_in_goods_sum_mart
after insert OR update or delete
on pract_functions.goods
for each row
execute procedure new_goods_in_goods_sum_mart();
```
4. Произвел вставку данных в таблицы sales и goods. Для вставки воспользовался скриптом из материалов к ДЗ. Код создания ниже:
```SQL
-- Вставка в таблицу goods
insert into pract_functions.goods (goods_id, goods_name, goods_price)
values (1,'Спички хозайственные', .50),
       (2,'Автомобиль Ferrari FXX K', 185000000.01);

-- вставка в таблицу sales
insert into pract_functions.sales (goods_id, sales_qty)
values 	(1, 10),
          (1, 1),
          (1, 120),
          (2, 1);
```
5. Выполнил select для всех трех таблиц
```SQL
-- Вставка в таблицу goods
select * from pract_functions.goods g;
select * from pract_functions.sales s;
select * from  pract_functions.goods_sum_mart gsm;
```
**Результат**

| goods_id  | goods_name               | goods_price  |
|-----------|--------------------------|--------------|
| 1         | Спички хозайственные     | 0.50         |
| 2         | Автомобиль Ferrari FXX K | 185000000.01 |

| sales_id  | goods_id  | sales_time                    | sales_qty  |
|-----------|-----------|-------------------------------|------------|
| 1         | 1         | 2024-12-11 10:37:01.234 +0700 | 10         |
| 2         | 1         | 2024-12-11 10:37:01.234 +0700 | 1          |
| 3         | 1         | 2024-12-11 10:37:01.234 +0700 | 120        |
| 4         | 2         | 2024-12-11 10:37:01.234 +0700 | 1          |

| goods_name               | sum_sale     |
|--------------------------|--------------|
| Спички хозайственные     | 65.50        |
| Автомобиль Ferrari FXX K | 185000000.01 |

6. Сравнил результат с запросом ниже:
```SQL
select
    G.goods_name,
    sum(G.goods_price * S.sales_qty)
from goods G
inner join sales S
    on S.goods_id = G.goods_id
group by G.goods_name;
```
**Результат:** Данные совпали
7. Далее проверил триггеры с помощью update и delete - проблем не возникло