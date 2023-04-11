### Первая сессия
postgres=# \set AUTOCOMMIT off
postgres=#  create table persons(id serial, first_name text, second_name text); 
insert into persons(first_name, second_name) values('ivan', 'ivanov'); 
insert into persons(first_name, second_name) values('petr', 'petrov'); commit;
CREATE TABLE
INSERT 0 1
INSERT 0 1
COMMIT
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
postgres=# begin;
BEGIN
postgres=* # insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
postgres=* #
### Вторая сессия
postgres=#  \set AUTOCOMMIT off
postgres=#  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
## новой строчки не увидел, так как уровень изоляции READ COMMITED, грязное чтение (неподтвержденные транзакции) недопускается.
##а в первой сессии транзакция не подтверждена.


### Первая сессия
postgres=* # commit; 
COMMIT
### Вторая сессия
postgres=* # select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
## новую строчку увидели, так как в первой сессии транзакцию завершили.
postgres=* # commit;
COMMIT


### Первая сессия
postgres=# set transaction isolation level repeatable read;
SET
postgres=* # insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
### Вторая сессия
postgres=# set transaction isolation level repeatable read;
SET
postgres=* #  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
## новой строчки не увидел, так как уровень изоляции REPEATABLE READ, грязное чтение (неподтвержденные транзакции) недопускаются.  
## в первой сессии транзакция не подтверждена. также данные изменённые другими транзакциями не видны в текущей транзакции. 


### Первая сессия
postgres=* # commit; 
COMMIT
### Вторая сессия
postgres=* #  select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
## новой строчки не увидел, так как уровень изоляции REPEATABLE READ, запрос видит срез данных на момент начала первого оператора в транзакции. 
## данные изменённые другими транзакциями не видны в текущей транзакции.

### Вторая сессия
postgres=* # commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
 
## новую строчкe увидели, так как в первой сессии транзакция была завершена. а во второй сессии мы начали новую транзакцию. 
