# Тема: Физический уровень PostgreSQL
1. Создал виртуальную машину, установил PostgreSQL 15, выдал доступ из вне
2. Проверил работоспособность 
3. Выполнил комманду:
```SQL
create table test(c1 text);

insert
into
    test
values('1');
```
4. Вышел из psql (\q) и остановил postgres коммандой: 
```
sudo -u postgres pg_ctlcluster 15 main stop
```
5. создайте новый диск для ВМ размером 10GB через Virtual Box Manager
6. Проинициализировал диск  и подмонтировал файловую систему согласно инструкции 
[Как разделить и отформатировать устройства хранения данных в Linux](https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux)
7. Перезагрузил ВМ - диск остался примонтирован
8. Сделал пользователя postgres владельцем /mnt/data коммандой:
```
chown -R postgres:postgres /mnt/data/
```
9. Перенес директорию /var/lib/postgres/15 в /mnt/data коммандой: 
```
mv /var/lib/postgresql/15 /mnt/data
```
10. Запустить кластер очевидно не вышло т.к. при запуске он ссылается на: 
```
data_directory = '/var/lib/postgresql/15/main'
```
Данный параметр находится в: postgresql.conf
```
/etc/postgresql/15/main/postgresql.conf
```
Поэтому в данном файле подменил путь на корректный и запустил кластер командой:
```
sudo -u postgres pg_ctlcluster 15 main start
```
11. Проверил содержимое данных. Все сохранилось
12. Вернул все дефолтное состояние