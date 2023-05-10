
**ОТВЕТ НА ПУНКТ 1**  
настройки  
log_lock_waits = on  
deadlock_timeout = 200ms  

запустил два разных сеанса  
в первом  
begin;  
SELECT pg_backend_pid();  
**20510**  
select * from accounts where acc_no = 1 for update;  

во втором  
begin;  
SELECT pg_backend_pid();  
**20613**  
select * from accounts where acc_no = 1 for update;  
  
и сеанс подвис.  
после выполнения в первом сеанса команды commit; во втором сеансе тоже можно было выполнить команду commit;  

в логе получил такую запись через 217.755 ms (выше мы установили время регистрации блокировки равной 200ms ), что процесс 20613  (второй сеанс) ждёт возможности захватить блокировку.  
ожидает он блокирующего процесса 20510 (первый сеанс).  
select * from accounts where acc_no = 1 for update;	 
process 20613 still waiting for ShareLock on transaction 333908 after 217.755 ms	Process holding the lock: 20510. Wait queue: 20613.  

после разблокировки получил вторую запись, что захватил блокировку через определённое время.  
select * from accounts where acc_no = 1 for update;	process 20613 acquired ShareLock on transaction 333908 after 4868.359 ms	 

**ОТВЕТ НА ПУНКТ 2**  
в первом сеансе  
begin; SELECT pg_backend_pid(); select * from accounts where acc_no = 1 for update;  
20510  
во втором сеансе  
begin; SELECT pg_backend_pid(); select * from accounts where acc_no = 2 for update;  
20613  
в третьем сеансе  
begin; SELECT pg_backend_pid(); select * from accounts where acc_no = 3 for update;  
21140  

в первом сеансе  
select * from accounts where acc_no = 2 for update;  
во втором сеансе  
select * from accounts where acc_no = 3 for update;  
в третьем сеансе  
select * from accounts where acc_no = 1 for update;  

и получаем deadlock; в логе видим запись  

deadlock detected	Process 21140 waits for ShareLock on transaction 333922; blocked by process 20510.  
Process 20510 waits for ShareLock on transaction 333923; blocked by process 20613.  
Process 20613 waits for ShareLock on transaction 333924; blocked by process 21140.  
Process 21140: select * from accounts where acc_no = 1 for update;  
Process 20510: select * from accounts where acc_no = 2 for update;  
Process 20613: select * from accounts where acc_no = 3 for update;	while locking tuple (0,6) in relation "accounts"  

интерпретировать её можно так:  
 процесс 21140(третий сеанс) заблокирован процессом 20510 (первый сеанс).  
 процесс 20510(первый сеанс) заблокирован процессом 20613 (вторым сеансом).  
 процесс 20613(второй сеанс) заблокирован процессом 21140 (третьим сеансом). и ctid строки в таблице   
 
который можно найти так  
 select ctid, * from accounts ;  
 ctid  | acc_no | amount  
-------+--------+---------  
 (0,2) |      2 | 2000.00  
 (0,3) |      3 | 3000.00  
 (0,6) |      1 | 1102.00  

**ОТВЕТ НА ПУНКТ 3**  
в первом сеансе  
begin;SELECT txid_current(),pg_backend_pid(); update accounts set amount = amount + 1 where acc_no = 1;  
 txid_current | pg_backend_pid  
--------------+----------------  
       333971 |          20510  
во втором сеансе  
begin;SELECT txid_current(),pg_backend_pid(); update accounts set amount = amount + 1 where acc_no = 1;  
 txid_current | pg_backend_pid   
--------------+----------------  
       333972 |          20613  
в третьем сеансе  
begin;SELECT txid_current(),pg_backend_pid(); update accounts set amount = amount + 1 where acc_no = 1;  
 txid_current | pg_backend_pid  
--------------+----------------  
       333973 |          21140  
 
смотрим блокировки вот так  
SELECT pid, locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks  where pid in (20510,20613,21140) order by pid  

 -pid--|---locktype----|-relation-|-virtxid-|--xid---|-------mode-------|granted  
 20510 | virtualxid----|----------| 3/28127 |--------| ExclusiveLock----| t  
 20510 | transactionid|----------|---------| 333974 | ExclusiveLock----| t  
 20510 | relation------| accounts |---------|--------| RowExclusiveLock-| t  
 20613 | transactionid|----------|---------| 333974 | ShareLock--------| f  
 20613 | tuple---------| accounts |---------|--------| ExclusiveLock----| t  
 20613 | transactionid|----------|---------| 333975 | ExclusiveLock----| t  
 20613 | relation------| accounts |---------|--------| RowExclusiveLock | t  
 20613 | virtualxid----|----------| 7/1851  |--------| ExclusiveLock----| t  
 21140 | virtualxid----|----------| 8/779   |--------| ExclusiveLock----| t  
 21140 | transactionid|----------|---------| 333976 | ExclusiveLock----| t  
 21140 | tuple---------| accounts |---------|--------| ExclusiveLock----| f  
 21140 | relation -----| accounts |---------|--------| RowExclusiveLock-| t  


видим что первый сеанс получил запрошенные блокировки.  
второй сеанс получил запрошенные блокировки, ему также пришлось запросить блокировку версии строки, и блокировку транзакции первого сеанса  
третий сеанс и последующие уже будут зависат на блокировки версии строки  

**ОТВЕТ НА ПУНКТ 4**  

например один запрос будет обновлять таблицу по возрастанию по id, второй запрос по убыванию по id.  
создаем таблицу и заполняем записями. если сервер мощный, то можно записей побольше вставить. главное успеть запустить второй сеанс до окончания первого.  
create table x ( y int , z int );  
INSERT INTO x  SELECT i,i  FROM generate_series(1, 1000000) AS t(i);  

в первом сеансе  
begin; update x SET  z = z + 1 FROM  (  select y from  x ORDER BY y asc ) d where   x.y = d.y;  
 
во втором сеансе  
begin; update x SET  z = z + 1 FROM  (  select y from  x ORDER BY y desc ) d where   x.y = d.y;  

ловим взаимоблокировку    

ERROR:  deadlock detected  
DETAIL:  Process 20510 waits for ShareLock on transaction 333954; blocked by process 20613.  
Process 20613 waits for ShareLock on transaction 333953; blocked by process 20510.  
HINT:  See server log for query details.  
CONTEXT:  while updating tuple (3258,40) in relation "x"  


