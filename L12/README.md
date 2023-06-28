--решил секционировать по полю flight_id, секции получаются немного разные по размеру, из-за разного количества проданных билетов на рейсы, но это не критично.   
create extension timescaledb;  
--создаем новую таблицу со структурой как у старой  
CREATE TABLE bookings.ticket_flights1 (LIKE bookings.ticket_flights INCLUDING DEFAULTS INCLUDING CONSTRAINTS EXCLUDING INDEXES);  
--выбираем размер секции. подбирается оптимальный размер удовлетворющий производительности сервера.  
SELECT create_hypertable('bookings.ticket_flights1', 'flight_id', chunk_time_interval => 5000);  
--переливаются данные  
INSERT INTO bookings.ticket_flights1 SELECT * FROM bookings.ticket_flights;  
--подменяем таблицы  
alter table bookings.ticket_flights rename to ticket_flights_old;  
alter table bookings.ticket_flights1 rename to ticket_flights;  
--смотрим секции  
SELECT show_chunks('bookings.ticket_flights');  

--пример запроса  
explain (analyze,verbose,buffers)  
select * from bookings.ticket_flights  
where flight_id = 30000  

Bitmap Heap Scan on _timescaledb_internal._hyper_1_1_chunk  (cost=1.66..38.69 rows=34 width=32) (actual time=0.028..0.032 rows=7 loops=1)  
  Output: _hyper_1_1_chunk.ticket_no, _hyper_1_1_chunk.flight_id, _hyper_1_1_chunk.fare_conditions, _hyper_1_1_chunk.amount  
  Recheck Cond: (_hyper_1_1_chunk.flight_id = 30000)  
  Heap Blocks: exact=1  
  Buffers: shared hit=3  
  ->  Bitmap Index Scan on _hyper_1_1_chunk_ticket_flights1_flight_id_idx  (cost=0.00..1.65 rows=34 width=0) (actual time=0.017..0.017 rows=7 loops=1)  
        Index Cond: (_hyper_1_1_chunk.flight_id = 30000)  
        Buffers: shared hit=2  
Query Identifier: -8901897068090214047  
Planning:  
  Buffers: shared hit=23  
Planning Time: 1.058 ms  
Execution Time: 0.080 ms  
