
**ответ на 16**  
postgres=# \c testdb testread  
You are now connected to database "testdb" as user "testread".  
testdb=> select * from t1;  
ERROR:  permission denied for table t1  

**ответ на 17**  
не получилось так как t1 находится в схеме public. а мы дали права на таблицы в схеме testnm  

**ответ на 18**  
чтобы получилось достаточно выдать права 
grant select on all tables  in  schema public to readonly;  
права USAGE, CREATE на схему public не обязательно, так как по умолчанию эти права есть.  

**ответ на 21**  
при создании таблицы, если не указана схема в имени, то таблица создается  в схеме как имя пользователя (если такая схема есть)  
либо в схеме public  

**ответ на 29**  
не получилось, так как таблица создалась после выдачи прав в пункте 10.  
чтобы права выдавались автоматически надо создать правило DEFAULT PRIVILEGES  

**ответ на 30**  
ALTER DEFAULT PRIVILEGES FOR ROLE postgres IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;   
и применить права на уже существующие таблицы  
grant select on all tables  in  schema testnm to readonly ;  

**ответ на 35**  
получилось. создалась таблица в схеме public. как и писал в ответе 18 по умолчанию есть права  USAGE, CREATE на схему public  

**ответ на 36**  
REVOKE USAGE,CREATE on schema public from public;  

**ответ на 37**  
я просто это знал из документации  

**ответ на 39**  
не получилось. так как нет ни схемы с именем пользователя, ни схема public недоступна  
