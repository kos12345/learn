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
