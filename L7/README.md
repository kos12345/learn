**Ответ на первый вопрос**  
checkpoint_timeout = 30s  
**Ответ на второй вопрос**  
--запуск pgbench  
/usr/pgsql-15/bin/pgbench -i postgres  
/usr/pgsql-15/bin/pgbench -c8 -P 60 -T 600 -U postgres postgres  
**Ответ на третий вопрос**  
перед запуском теста запомнил текущий lsn  
select pg_current_wal_lsn()     --**0/177D868**  
после окончания теста посчитал объем  
select pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(),'0/177D868'))  
получилось 559 MB - но это объем записанный в wal-журнал  
  
в файлы бд сколько скинуто информации можно посчитать так  
таблица pg_stat_bgwriter  buffers_checkpoint  / (1024x1024 /8192) (block_size) и получится что записалось процессом  340mb  
разделив на количество чекпойнтов за 10 минут (20 штук) получим в среднем около 17 mb на один чекпойнт  

**Ответ на четвертый вопрос**  
статистика показывает checkpoints_req 0  
то есть ни одного дополнительного чекпоинта не было вызвано  
это потому, что max_wal_size у нас по умолчанию 1Gb. и наш процесс тестирования не потребил ни разу этот объем за 30 секунд  

**Ответ на пятый вопрос**  
в синхронном режиме  
tps = 824.973821 (without initial connection time)  
в асинхронном  
tps = 1498.490661 (without initial connection time)  
  
это происходит потому что в режиме асинхронного подтверждения сервер сообщает об успешном завершении сразу, как только транзакция будет завершена логически,прежде чем сгенерированные записи WAL фактически будут записаны на диск.  
Это и привело к увеличению производительности.  

**Ответ на шестой вопрос**  
субд инициализирована вот такой командой.  
/usr/pgsql-15/bin/initdb -D /var/lib/pgsql/15/data --data-checksums  
подготавливаем бд, таблицу и заполняем данными   
create database test;  
create table test ( id int, txt varchar );  
INSERT INTO test  SELECT i,left(md5(random()::text),5)   FROM generate_series(1, 10000) AS t(i);  
 
запоминаем путь до таблицы в ОС  
SELECT pg_relation_filepath('test');  

находим в середине данные. которые затем сломаем в файле  
select ctid, * from test  
where ctid  in ('(47,19)','(47,20)','(47,21)','(47,18)')  

systemctl stop postgresql-15  
находим данные в файле. и меняем один символ какой нибудь.  
systemctl start postgresql-15  

select ctid, * from test  
выдает ошибку  
SQL Error [XX001]: ERROR: invalid page in block 10 of relation base/16412/16418  
page verification failed, calculated checksum 20239 but expected 40736  

если поставим такие параметры, то сможем  читать таблицу, обходя сбойную страницу  
ignore_checksum_failure = off  


