**На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.**  
использую ВМ с ОС rhel.  
/usr/pgsql-15/bin/initdb -D /var/lib/pgsql/15/data  
echo listen_addresses = \'*\' >> /var/lib/pgsql/15/data/postgresql.conf  
echo port = \'5432\' >> /var/lib/pgsql/15/data/postgresql.conf  
echo wal_level = \'logical\' >> /var/lib/pgsql/15/data/postgresql.conf  
echo host all all 0.0.0.0/0 md5 >> /var/lib/pgsql/15/data/pg_hba.conf  
echo host replication all 0.0.0.0/0 md5 >> /var/lib/pgsql/15/data/pg_hba.conf  
/usr/pgsql-15/bin/pg_ctl start -D /var/lib/pgsql/15/data/  

psql -p5432  
create database db1;  
\c db1  
create table test as  select  generate_series(1,10) as id, '1-' || md5(random()::text)::char(10) as describe;  
CREATE TABLE test2 (LIKE test);  
CREATE PUBLICATION test_pub FOR TABLE test;  
alter user postgres password 'password';  

**На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.**  
mkdir /var/lib/pgsql/15/data2  
/usr/pgsql-15/bin/initdb -D /var/lib/pgsql/15/data2  
cp /var/lib/pgsql/15/data/postgresql.conf /var/lib/pgsql/15/data2/postgresql.conf  
cp /var/lib/pgsql/15/data/pg_hba.conf /var/lib/pgsql/15/data2/pg_hba.conf  
sed -i 's/5432/5433/g' /var/lib/pgsql/15/data2/postgresql.conf  
/usr/pgsql-15/bin/pg_ctl start -D /var/lib/pgsql/15/data2/  

psql -p 5433  
create database db1;  
\c db1  
create table test2 as  select  generate_series(1,10) as id, '2-' || md5(random()::text)::char(10) as describe;  
CREATE TABLE test (LIKE test2);  
CREATE PUBLICATION test_pub FOR TABLE test2;  
alter user postgres password 'password2';  
CREATE SUBSCRIPTION test_sub CONNECTION 'host=localhost port=5432 user=postgres password=password dbname=db1'  PUBLICATION test_pub WITH (copy_data = true);  

psql -p5432  
\c db1  
CREATE SUBSCRIPTION test_sub CONNECTION 'host=localhost port=5433 user=postgres password=password2 dbname=db1'  PUBLICATION test_pub WITH (copy_data = true);  

**3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).**  
mkdir /var/lib/pgsql/15/data3  
/usr/pgsql-15/bin/initdb -D /var/lib/pgsql/15/data3  
cp /var/lib/pgsql/15/data/postgresql.conf /var/lib/pgsql/15/data3/postgresql.conf;  
cp /var/lib/pgsql/15/data/pg_hba.conf /var/lib/pgsql/15/data3/pg_hba.conf  
sed -i 's/5432/5434/g' /var/lib/pgsql/15/data3/postgresql.conf  
/usr/pgsql-15/bin/pg_ctl start -D /var/lib/pgsql/15/data3/  

psql -p 5434  
create database db1;  
\c db1  
первая проблема - выяснить структуру таблиц    
create table test (id int, describe text);  
create table test2 (id int, describe text);  
вторая проблема - совпадение имен подписок. 
CREATE SUBSCRIPTION test_sub1 CONNECTION 'host=localhost port=5433 user=postgres password=password2 dbname=db1'  PUBLICATION test_pub WITH (copy_data = true);  
could not create replication slot "test_sub": ERROR:  replication slot "test_sub" already exists  
https://www.postgresql.org/docs/current/sql-createsubscription.html  
The default is to use the name of the subscription for the slot name.  

как выход использовать slot_name или по другому называть подписки  
CREATE SUBSCRIPTION test_sub1 CONNECTION 'host=localhost port=5433 user=postgres password=password2 dbname=db1'  PUBLICATION test_pub WITH (copy_data = true,slot_name= srv3_test );  
CREATE SUBSCRIPTION test_sub2 CONNECTION 'host=localhost port=5432 user=postgres password=password dbname=db1'  PUBLICATION test_pub WITH (copy_data = true,slot_name= srv3_test2 );  

**реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3.**  
mkdir /var/lib/pgsql/15/data4  
chmod 0700 /var/lib/pgsql/15/data4  
pg_basebackup -p 5434 -R -D /var/lib/pgsql/15/data4  

sed -i 's/5434/5435/g' /var/lib/pgsql/15/data4/postgresql.conf  
echo 'hot_standby = on' >> /var/lib/pgsql/15/data4/postgresql.conf  
/usr/pgsql-15/bin/pg_ctl start -D /var/lib/pgsql/15/data4/  
никаких проблем не возникло.  
