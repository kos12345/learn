
**Создаем БД, схему и в ней таблицу.**  
create database test; --создали бд  
\c test --переключились в контекст этой бд  
create schema test; --создали схему  
create table test.table1 ( id serial, notes varchar); --создали таблицу в этой схеме  

**Заполним таблицы автосгенерированными 100 записями.**  
INSERT INTO test.table1( notes) SELECT left(md5(random()::text),20) FROM generate_series(1, 100);  --заполнили таблицу  

**Под линукс пользователем Postgres создадим каталог для бэкапов**  
su - postgres -- переключились под пользователя postgres  
mkdir /var/lib/pgsql/backups --создали директорию для бэкапов  

**Сделаем логический бэкап используя утилиту COPY**  
\copy test.table1 to '/var/lib/pgsql/backups/table1.sql'; 

**Восстановим в 2 таблицу данные из бэкапа.**  
create table test.table2 ( id serial, notes varchar);  --сначала создали вторую таблицу. так как в бэкапе нет скрипта создания таблицы  
\copy test.table2  from '/var/lib/pgsql/backups/table1.sql' --восстанавливаем  


**Используя утилиту pg_dump создадим бэкап с оглавлением в кастомном сжатом формате 2 таблиц**  
pg_dump -d test -Fd -f /var/lib/pgsql/backups/backup1  
-Fd ключ для создания бэкапа с оглавлением  
-d имя бд, которую бэкапим  
-f путь для бэкапа  

**Используя утилиту pg_restore восстановим в новую БД только вторую таблицу!**  
psql -c "create database test2;" --первым шагом создадим новую бд  

сначала пробуем вот такую команду и понимаем, что она не работает. так как схемы test нет  
pg_restore -d test2 -n test -t table2 -Fd /var/lib/pgsql/backups/backup1  

есть два варианта.
1. вариант
сначала создать схему test в бд test2 и запустить pg_restore приведённый выше

2. вариант
pg_restore -l -Fd /var/lib/pgsql/backups/backup1 > /var/lib/pgsql/backups/list --подготавливаем список существующих объектов в бэкапе.  
vi /var/lib/pgsql/backups/list --редактируем список для восстановления. оставляем только схему test и таблицу table2.  
pg_restore -d test2 -Fd /var/lib/pgsql/backups/backup1 -L /var/lib/pgsql/backups/list --восстанавливаем  

postgres=# \c test2  
You are now connected to database "test2" as user "postgres".  
test2=# \dt test.*  
         List of relations   
 Schema |  Name  | Type  |  Owner  
--------+--------+-------+----------  
 test   | table2 | table | postgres  

