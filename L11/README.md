**1 вариант:**  
**1.2.Создать индекс к какой-либо из таблиц вашей БД.Прислать текстом результат команды explain,
в которой используется данный индекс**  
--запуск без индекса
explain(analyze,buffers)  
select * from test where id = '555555'  
Gather  (cost=1000.00..12578.49 rows=1 width=17) (actual time=156.360..158.831 rows=1 loops=1)  
  Buffers: shared hit=6370  
  ->  Parallel Seq Scan on test  (cost=0.00..11578.39 rows=1 width=17) (actual time=118.030..138.046 rows=0 loops=3)  
        Filter: (id = 555555)  
        Rows Removed by Filter: 333336  
        Buffers: shared hit=6370  
Planning Time: 0.109 ms  
Execution Time: 158.860 ms  
--создаем индекс  
create index i_test on test using btree (id);  
select * from test where id = '555555'  
Index Scan using i_test on test  (cost=0.42..2.64 rows=1 width=17) (actual time=0.037..0.038 rows=1 loops=1)  
  Index Cond: (id = 555555)  
  Buffers: shared hit=1 read=3  
Planning:  
  Buffers: shared hit=15 read=1  
Planning Time: 0.203 ms  
Execution Time: 0.052 ms  

**3.Реализовать индекс для полнотекстового поиска**  
--таблица и индекс  
create table test1 ( id int,dt date, txt text);  
CREATE INDEX i_fts ON public.test1 USING gin (to_tsvector('russian'::regconfig, txt));  

explain (analyze,buffers)  
select * from test1  where to_tsvector('russian', txt) @@ to_tsquery('текст');  
Bitmap Heap Scan on test1  (cost=9.62..667.85 rows=500 width=39) (actual time=5.216..15.154 rows=16558 loops=1)  
  Recheck Cond: (to_tsvector('russian'::regconfig, txt) @@ to_tsquery('текст'::text))  
  Heap Blocks: exact=876  
  Buffers: shared hit=882  
  ->  Bitmap Index Scan on i_fts  (cost=0.00..9.50 rows=500 width=0) (actual time=4.864..4.864 rows=16558 loops=1)  
        Index Cond: (to_tsvector('russian'::regconfig, txt) @@ to_tsquery('текст'::text))  
        Buffers: shared hit=6  
Planning:  
  Buffers: shared hit=1  
Planning Time: 0.198 ms  
Execution Time: 18.465 ms  

**4.Реализовать индекс на часть таблицы или индекс на поле с функцией**  
create index i_data on test1 using btree (id) where dt > '2019-05-01';  
explain (analyze,buffers)  
select * from test1 where dt = '2019-06-01' limit 10;  
Limit  (cost=0.29..53.23 rows=10 width=39) (actual time=0.026..0.498 rows=10 loops=1)  
  Buffers: shared hit=21  
  ->  Index Scan using i_data on test1  (cost=0.29..1752.68 rows=331 width=39) (actual time=0.025..0.493 rows=10 loops=1)  
        Filter: (dt = '2019-06-01'::date)  
        Rows Removed by Filter: 1080  
        Buffers: shared hit=21  
Planning Time: 0.179 ms  
Execution Time: 0.516 ms  

**5.Создать индекс на несколько полей**  
create index i_id_data on test1 using btree (id,dt);  
explain (analyze,buffers)  
select id,dt from test1 where  id > 1000;  
Index Only Scan using i_id_data on test1  (cost=0.29..2034.79 rows=99034 width=8) (actual time=0.163..50.185 rows=99000 loops=1)  
  Index Cond: (id > 1000)  
  Heap Fetches: 0  
  Buffers: shared hit=2 read=272  
Planning:  
  Buffers: shared hit=3  
Planning Time: 0.193 ms  
Execution Time: 64.202 ms  

**2 вариант:**  
**1.Реализовать прямое соединение двух или более таблиц**  
explain (analyze,buffers)  
select * from test as t join test1 as t2 on t.id = t2.id;  
Merge Join  (cost=0.81..6527.44 rows=100219 width=56) (actual time=25.616..96.961 rows=50001 loops=1)  
  Merge Cond: (t2.id = t.id)  
  Buffers: shared hit=1560 read=272  
  ->  Index Scan using i_id on test1 t2  (cost=0.29..2679.99 rows=100000 width=39) (actual time=0.011..31.653 rows=100000 loops=1)  
        Buffers: shared hit=964 read=272  
  ->  Index Scan using i_test on test t  (cost=0.42..24390.17 rows=1000010 width=17) (actual time=0.147..15.661 rows=50002 loops=1)  
        Buffers: shared hit=596  
Planning:  
  Buffers: shared hit=160 read=4  
Planning Time: 0.550 ms  
Execution Time: 101.783 ms  

**2.Реализовать левостороннее (или правостороннее) соединение двух или более таблиц**  
explain (analyze,buffers)  
select * from test as t left join test1 as t2 on t.id = t2.id;  
Merge Right Join  (cost=0.81..30822.38 rows=1000010 width=56) (actual time=30.102..662.840 rows=950001 loops=1)  
  Merge Cond: (t2.id = t.id)  
  Buffers: shared hit=10023  
  ->  Index Scan using i_id on test1 t2  (cost=0.29..2679.99 rows=100000 width=39) (actual time=0.014..34.194 rows=100000 loops=1)  
        Buffers: shared hit=1236  
  ->  Index Scan using i_test on test t  (cost=0.42..24390.17 rows=1000010 width=17) (actual time=0.186..284.193 rows=950001 loops=1)  
        Buffers: shared hit=8787  
Planning:  
  Buffers: shared hit=147  
Planning Time: 0.612 ms  
Execution Time: 753.652 ms  

**3.Реализовать кросс соединение двух или более таблиц**  
explain (analyze,buffers)  
select * from test as t cross join test1 as t2;  
Nested Loop  (cost=0.00..10253.53 rows=808101 width=56) (actual time=0.024..334.622 rows=808101 loops=1)  
  Buffers: shared hit=71  
  ->  Seq Scan on test1 t2  (cost=0.00..150.01 rows=8001 width=39) (actual time=0.007..2.394 rows=8001 loops=1)  
        Buffers: shared hit=70  
  ->  Materialize  (cost=0.00..2.51 rows=101 width=17) (actual time=0.000..0.011 rows=101 loops=8001)  
        Buffers: shared hit=1  
        ->  Seq Scan on test t  (cost=0.00..2.01 rows=101 width=17) (actual time=0.006..0.012 rows=101 loops=1)  
              Buffers: shared hit=1  
Planning:  
  Buffers: shared hit=19 dirtied=1  
Planning Time: 0.119 ms  
Execution Time: 410.951 ms  

**4.Реализовать полное соединение двух или более таблиц**  
explain (analyze,buffers)  
select * from test as t full join test1 as t2 on t.id = t2.id;  
Hash Full Join  (cost=3.27..184.30 rows=8001 width=56) (actual time=0.141..9.368 rows=8102 loops=1)  
  Hash Cond: (t2.id = t.id)  
  Buffers: shared hit=71  
  ->  Seq Scan on test1 t2  (cost=0.00..150.01 rows=8001 width=39) (actual time=0.008..2.427 rows=8001 loops=1)  
        Buffers: shared hit=70  
  ->  Hash  (cost=2.01..2.01 rows=101 width=17) (actual time=0.104..0.105 rows=101 loops=1)  
        Buckets: 1024  Batches: 1  Memory Usage: 13kB  
        Buffers: shared hit=1  
        ->  Seq Scan on test t  (cost=0.00..2.01 rows=101 width=17) (actual time=0.011..0.038 rows=101 loops=1)  
              Buffers: shared hit=1  
Planning:  
  Buffers: shared hit=27  
Planning Time: 0.382 ms  
Execution Time: 10.839 ms  

**5.Реализовать запрос, в котором будут использованы разные типы соединений**  
explain (analyze,buffers)  
select * from test as t  
left join test1 as t1 on t1.id = t.id  
full join test2 as t2 on t2.id =t1.id;  
Hash Full Join  (cost=66.01..221.12 rows=11494 width=93) (actual time=0.198..0.215 rows=15 loops=1)  
  Hash Cond: (t2.id = t1.id)  
  Buffers: shared hit=4  
  ->  Seq Scan on test2 t2  (cost=0.00..22.70 rows=1270 width=36) (actual time=0.006..0.008 rows=10 loops=1)  
        Buffers: shared hit=1  
  ->  Hash  (cost=43.39..43.39 rows=1810 width=57) (actual time=0.186..0.188 rows=11 loops=1)  
        Buckets: 2048  Batches: 1  Memory Usage: 17kB  
        Buffers: shared hit=3  
        ->  Hash Left Join  (cost=6.50..43.39 rows=1810 width=57) (actual time=0.169..0.179 rows=11 loops=1)  
              Hash Cond: (t.id = t1.id)  
              Buffers: shared hit=3  
              ->  Seq Scan on test t  (cost=0.00..28.10 rows=1810 width=17) (actual time=0.006..0.008 rows=11 loops=1)  
                    Buffers: shared hit=1  
              ->  Hash  (cost=4.00..4.00 rows=200 width=40) (actual time=0.155..0.156 rows=200 loops=1)  
                    Buckets: 1024  Batches: 1  Memory Usage: 23kB  
                    Buffers: shared hit=2  
                    ->  Seq Scan on test1 t1  (cost=0.00..4.00 rows=200 width=40) (actual time=0.008..0.058 rows=200 loops=1)  
                          Buffers: shared hit=2  
Planning:  
  Buffers: shared hit=8  
Planning Time: 0.402 ms  
Execution Time: 0.255 ms  

**К работе приложить структуру таблиц, для которых выполнялись соединения**  
CREATE TABLE public.test (	id int4 NULL,	"describe" text NULL);  
CREATE INDEX i_test ON public.test USING btree (id);  

CREATE TABLE public.test1 (	id int4 NULL,	dt date NULL,	txt text NULL);  
CREATE INDEX i_data ON public.test1 USING btree (id) WHERE (dt > '2019-05-01'::date);  
CREATE INDEX i_fts ON public.test1 USING gin (to_tsvector('russian'::regconfig, txt));  
CREATE INDEX i_id ON public.test1 USING btree (id);  

CREATE TABLE public.test2 (	id int4 NULL,	"describe" text NULL);  

Проблемы возникли только на cross join. В таблицах изначально было по 100тысяч записей. и я не дожидался ответа результата.  
Надо было либо добавлять ресурсы и править настройки в postgresql.conf либо уменьшить количество строк. Я выбрал второй вариант.  



