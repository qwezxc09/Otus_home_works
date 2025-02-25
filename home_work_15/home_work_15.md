# Секционирование
1. Загрузил тестовую базу demo выполнив комманды (были ошибки поэтому выполнял комманды по одной а ни как в примере):
```CMD
sudo wget https://edu.postgrespro.ru/demo_small.zip
sudo apt install unzip
sudo unzip demo_small.zip
chmod 777 /home/renegadeadm/ - [/home renegadeadm это домашний каталог]
sudo -u postgres psql -d postgres -f /home/renegadeadm/demo_small.sql -c 'alter database demo set search_path to bookings' 
```
2. Посмотрел размер всех таблиц и выбрал наиболее подходящуюю ticket_flights.

| schema   | table_name      | total_size  | table_size  | index_size  |
|----------|-----------------|-------------|-------------|-------------|
| bookings | ticket_flights  | 109 MB      | 68 MB       | 41 MB       |
| bookings | boarding_passes | 81 MB       | 33 MB       | 47 MB       |
| bookings | tickets         | 59 MB       | 48 MB       | 11 MB       |
| bookings | bookings        | 19 MB       | 13 MB       | 5816 kB     |
| bookings | flights         | 4944 kB     | 3136 kB     | 1808 kB     |
| bookings | routes          | 152 kB      | 136 kB      | 16 kB       |
| bookings | seats           | 144 kB      | 64 kB       | 80 kB       |
| bookings | airports        | 64 kB       | 16 kB       | 48 kB       |
| bookings | aircrafts       | 32 kB       | 8192 bytes  | 24 kB       |

**Атрибутный состав таблицы ticket_flights**

| column_name     | data_type         |
|-----------------|-------------------|
| flight_id       | integer           |
| amount          | numeric           |
| ticket_no       | character         |
| fare_conditions | character varying |


3. Создал дубликат таблицы ticket_flights, под названием ticket_flights_part, у которой была возможность секционирования по списку (атрибут fare_conditions, пришлось добавить данный атрибут в составной ключ)

```SQL
 --1 Создание таблицы bookings.ticket_flights_part, по подобию bookings.ticket_flights, кроме наличия секционирования

create table bookings.ticket_flights_part (
                                              ticket_no bpchar(13) not null,
                                              flight_id int4 not null,
                                              fare_conditions varchar(10) not null,
                                              amount numeric(10,2) not null,
                                              constraint ticket_flights_part_amount_check check ((amount >= (0)::numeric)),
                                              constraint ticket_flights_part_fare_conditions_check check (((fare_conditions)::text = any (array[('Economy'::character varying)::text,
                                                  ('Comfort'::character varying)::text,
                                                  ('Business'::character varying)::text]))),
	constraint ticket_flights_part_pkey primary key (ticket_no,flight_id,fare_conditions)
										) partition by LIST (fare_conditions);

--2 Создание ограничений на таблицу bookings.ticket_flights_part по подобию таблицы ticket_flights

alter table bookings.ticket_flights_part add constraint ticket_flights_part_flight_id_fkey foreign key (flight_id) references bookings.flights(flight_id);

alter table bookings.ticket_flights_part add constraint ticket_flights_part_ticket_no_fkey foreign key (ticket_no) references bookings.tickets(ticket_no);

--3 Создание секций в таблице bookings.ticket_flights_part

create table bookings.ticket_flights_part_business partition of bookings.ticket_flights_part
    for values in ('Business');

create table bookings.ticket_flights_part_comfort partition of bookings.ticket_flights_part
    for values in ('Comfort');

create table bookings.ticket_flights_part_economy partition of bookings.ticket_flights_part
    for values in ('Economy');

--4 Заполнение данными таблицы bookings.ticket_flights_part из таблицы bookings.ticket_flights

insert
into
    ticket_flights_part (ticket_no ,
                         flight_id ,
                         fare_conditions,
                         amount)
select
    ticket_no ,
    flight_id ,
    fare_conditions,
    amount
from
    ticket_flights;
```
4. По результатам выяснилось, что таблица для мест с классом эконом по прежнему ззанимает много места

| schema   | table_name                   | total_size  | table_size | index_size  |
|----------|------------------------------|-------------|------------|-------------|
| bookings | ticket_flights_part_economy  | 119 MB      | 60 MB      | 59 MB       |
| bookings | ticket_flights_part_comfort  | 2296 kB     | 1160 kB    | 1136 kB     |
| bookings | ticket_flights_part_business | 14 MB       | 7184 kB    | 7032 kB     |

Поэтому решил разбить ее по хэшу по столбцу ticket_no (т.к. это поле с самыми уникальными значениями) для этого удалил текущую таблицу и выполнил предыдущие действия, переделав 3ий пункт
```SQL
	--3 Создание секций в таблице bookings.ticket_flights_part

create table bookings.ticket_flights_part_business partition of bookings.ticket_flights_part
    for values in ('Business');

create table bookings.ticket_flights_part_comfort partition of bookings.ticket_flights_part
    for values in ('Comfort');

create table bookings.ticket_flights_part_economy partition of bookings.ticket_flights_part
    for values in ('Economy') PARTITION BY HASH (ticket_no);

create table ticket_flights_part_economy_1 partition of ticket_flights_part_economy
    for values with (modulus 4, remainder 0);

create table ticket_flights_part_economy_2 partition of ticket_flights_part_economy
    for values with (modulus 4, remainder 1);

create table ticket_flights_part_economy_3 partition of ticket_flights_part_economy
    for values with (modulus 4, remainder 2);

create table ticket_flights_part_economy_4 partition of ticket_flights_part_economy
    for values with (modulus 4, remainder 3);
```
В итоге получил результат приведенный ниже:

| schema   | table_name                    | total_size  | table_size  | index_size  |
|----------|-------------------------------|-------------|-------------|-------------|
| bookings | ticket_flights_part_economy_4 | 29 MB       | 15 MB       | 14 MB       |
| bookings | ticket_flights_part_economy_3 | 29 MB       | 15 MB       | 14 MB       |
| bookings | ticket_flights_part_economy_2 | 29 MB       | 15 MB       | 14 MB       |
| bookings | ticket_flights_part_economy_1 | 29 MB       | 15 MB       | 14 MB       |
| bookings | ticket_flights_part_comfort   | 2304 kB     | 1160 kB     | 1144 kB     |
| bookings | ticket_flights_part_business  | 14 MB       | 7184 kB     | 7040 kB     |
