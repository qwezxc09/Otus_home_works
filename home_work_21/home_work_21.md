# Виды и устройство репликации в PostgreSQL. Практика применения 
1. Создал 4 виртуальные машины. Сконфигурировал файлы postgresql.conf, pg_hba.conf. ЧТобы можно было подключаться из вне. 
2. Выстовил первой и второй машине wal_level = logical:
```SQL
alter system set wal_level = logical;
```
3. Рестартанул posgres на всех ВМ:
```CMD
sudo systemctl restart postgresql
```
4. В базе первой, второй и третьей ВМ создал таблицы test и test2:
```SQL
create table test (id int);
create table test2 (id int);
```
5. Сделал публикацию таблицы test для 1-ой ВМ:
```SQL
create publication test_pub for table test;
```
6. Сделал публикацию таблицы test для 2-ой ВМ:
```SQL
create publication test2_pub for table test2;
```
7. Подписался на публикации первой ВМ на второй ВМ:
```SQL
create subscription test_sub 
connection 'host=89.169.130.118 port=5432 user=postgres password=123456 dbname=postgres' 
publication test_pub with (copy_data = true);
```
8. Подписался на публикации второй ВМ на певрой ВМ:
```SQL
create subscription test2_sub 
connection 'host=89.169.137.29 port=5432 user=postgres password=123456 dbname=postgres' 
publication test2_pub with (copy_data = true);
```
9. Подписался на публикации первой и второй ВМ на третьей ВМ:
```SQL
create subscription test3_sub 
connection 'host=host=89.169.130.118 port=5432 user=postgres password=123456 dbname=postgres' 
publication test_pub with (copy_data = true);
       
create subscription test4_sub 
connection 'host=host=89.169.137.29 port=5432 user=postgres password=123456 dbname=postgres' 
publication test2_pub with (copy_data = true);       
```
10. На четвертой ВМ остановил кластер, удалил папку main, и указал откуда брать бэкап:
```CMD
sudo systemctl stop postgresql
sudo rm -rf /var/lib/postgresql/15/main
sudo -u postgres pg_basebackup -h 89.169.137.120  -p 5432 -R -D /var/lib/postgresql/15/main
sudo systemctl start postgresql
```
1. По данным шагам проверил работоспособность с помощью инсертов на первой и второй ВМ, все работает. На каждой ВМ данные одинаковые