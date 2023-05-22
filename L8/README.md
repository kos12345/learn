ресурсы сервера 2CPU, 4GB RAM, SSD  
бд была инициализирована **без data_checksums**  
настройки  

random_page_cost = '1.1'  **настройка для SSD диска**  
fsync = off **отключение принудительного вызова fsync**  
full_page_writes = off   **отключение сохранения образа всей страницы в wal журнале**  
synchronous_commit = off   **отключение ожидания ответа о записи WAL журнала**  
effective_io_concurrency=200    **больше операций ввода/вывода будет пытаться выполнить параллельно PostgreSQL в отдельном сеансе**  
wal_level = minimal  **минимизировать запись в wal Журнал**  

вот эти настройки памяти особо не играли роли  
work_mem = 8MB  
shared_buffers = 1GB         - 25% от RAM  
effective_cache_size = 3GB  - 75% от RAM  


также все таблицы перевёл в режим UNLOGGED (в принципе часть настроек относящихся в записи WAL выше можно было не делать, так как нежурналирыемые таблицы не проходят через WAL журнал )  
alter table pgbench_accounts set UNLOGGED;  
alter table pgbench_branches set UNLOGGED;  
alter table pgbench_history set UNLOGGED;  
alter table pgbench_tellers set UNLOGGED;  

запуск осуществлялся командой  
/usr/pgsql-15/bin/pgbench -c8 -P 60 -T 600 -U postgres postgres  
дополнительные настройки -j в разумных пределах (2-8) и количество подключений -c (2-50) особо на ситуацию не влияли  

результаты tsp  
tps = 682.161603 (without initial connection time)  - без настроек + data_checksums = on  
tps = 1368.058354 (without initial connection time) - с настройкой synchronous_commit = off  + data_checksums = on  
tps = 1473.809830 (without initial connection time) - с остальными настройками  

как видим остальные настройки дали прирост дополнительный на уровне 7%.  
если тестировать на тысячах подключений, возможно дополнительные настройки и будут иметь смысл.  
в текущей ситуации достаточно было synchronous_commit = off сделать

