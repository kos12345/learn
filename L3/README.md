**проверяем запущенный кластер**  
#pg_lsclusters  
Ver Cluster Port Status Owner    Data directory              Log file  
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log  


**создал директорию, примонтировал диск в неё и переместил данные**  
mkdir /var/lib/postgresql/15/main1  
vi /etc/fstab  
mount -a  
mv /var/lib/postgresql/15/main/* /var/lib/postgresql/15/main1/  
 
**после старта посоветовало посмотреть  в journalctl -xeu postgresql @ 15-main.service**  
**в логе видим ошибку. означает что по существующему пути нет бд**  
pg_ctl: directory "/var/lib/postgresql/15/main" is not a database cluster directory  

**смотрим в конфиге путь , где должна находиться база данных**  
cat /etc/postgresql/15/main/postgresql.conf | grep data_dir  
data_directory = '/var/lib/postgresql/15/main'          # use data in another directory  

**исправляем конфиг на новый путь**  
sed -i 's/\\/var\\/lib\/postgresql\\/15\\/main/\\/var\\/lib\/postgresql\\/15\/main1/g' /etc/postgresql/15/main/postgresql.conf  

**после старта посоветовало посмотреть  в journalctl -xeu postgresql @ 15-main.service**  
**видим неправильные разрешения на директорию**  
data directory "/var/lib/postgresql/15/main1" has invalid permissions  

**исправляем**  
chown postgres:postgres /var/lib/postgresql/15/main1  
chmod 0750 /var/lib/postgresql/15/main1  

**запускается успешно**  


## **перенос диска с базой данных**  
**на новой машине**  
rm -rf /var/lib/postgresql/15/main/*  
**монитруем в эту папку диск с первой машины**  
vi /etc/fstab  
mount -a  
**на всякий случай перевыдаём права**  
chown postgres:postgres /var/lib/postgresql/15/main  
chmod 0750 /var/lib/postgresql/15/main  
**запускаем кластер**  
pg_ctlcluster 15 main start  
**в логе будут ошибки о неккоректной остановке базы данных. мы же отмонтировали диск не выключив субд**  
**кластер в итоге запустится успешно**  
2023-04-24 21:22:47.315 MSK [4991] LOG:  starting PostgreSQL 15.2 (Ubuntu 15.2-1.pgdg22.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 11.3.0-1ubuntu1~22.04) 11.3.0, 64-bit  
2023-04-24 21:22:47.315 MSK [4991] LOG:  listening on IPv4 address "127.0.0.1", port 5432  
2023-04-24 21:22:47.316 MSK [4991] LOG:  listening on Unix socket "/var/run/postgresql/.s.PGSQL.5432"  
2023-04-24 21:22:47.320 MSK [4994] LOG:  database system was interrupted; last known up at 2023-04-24 21:17:59 MSK  
2023-04-24 21:22:47.404 MSK [4994] LOG:  database system was not properly shut down; automatic recovery in progress  
2023-04-24 21:22:47.406 MSK [4994] LOG:  redo starts at 0/1562EC0  
2023-04-24 21:22:47.406 MSK [4994] LOG:  invalid record length at 0/1562EF8: wanted 24, got 0  
2023-04-24 21:22:47.406 MSK [4994] LOG:  redo done at 0/1562EC0 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00 s  
2023-04-24 21:22:47.409 MSK [4992] LOG:  checkpoint starting: end-of-recovery immediate wait  
2023-04-24 21:22:47.415 MSK [4992] LOG:  checkpoint complete: wrote 3 buffers (0.0%); 0 WAL file(s) added, 0 removed, 0 recycled; write=0.002 s, sync=0.001 s, total=0.007 s; sync files=2, longest=0.001 s, average=0.001 s; distance=0 kB, estimate=0 kB  
2023-04-24 21:22:47.419 MSK [4991] LOG:  database system is ready to accept connections  
