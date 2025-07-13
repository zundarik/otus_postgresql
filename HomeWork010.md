### Описание/Пошаговая инструкция выполнения домашнего задания:
- Настроить физическую репликацию между двумя нодами: мастер - слейв.
- Настроить каскадную репликацию со слейва на третью ноду.
- Настроить логическую репликацию таблицы с мастер ноды на четвертую ноду.

---
```
wsl
sudo -u postgres psql -d otus
```
```
otus=# select count(*) from student;
 count
-------
   180
(1 row)

select * from student;
 id |    fio
----+------------
  1 | 0a6c457efb
  2 | c01a7faa33
  3 | 90c381ba88
  4 | 0c9aacf4df
  5 | 06c5505e45
  6 | de3ce00efa
...
```
--- 
#### Проверка - кластер один, реплики нет
```
otus=# show wal_level;
 wal_level
-----------
 replica
(1 row)

pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
16  main    5432 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log

otus=# SELECT * FROM pg_stat_replication \gx
(0 rows)

otus=# SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 2/AA0263B8
(1 row)
```

--- 
#### Создание кластера
```
sudo pg_createcluster -d /var/lib/postgresql/16/main2 16 main2

pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
16  main    5432 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 down   postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
```

#### удаленние всех данный 2го кластера
```
sudo rm -rf /var/lib/postgresql/16/main2		
```

--- 
#### Физический бэкап
```
sudo -u postgres pg_basebackup -p 5432 -R -D /var/lib/postgresql/16/main2
16  main    5432 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 online postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
```

---
#### Проблема моей ошибки 
##### Помогла установка = 100 для max_connections и создание бэкапа заново
```
sudo pg_ctlcluster 16 main2 restart
Job for postgresql@16-main2.service failed because the service did not take the steps required by its unit configuration.
See "systemctl status postgresql@16-main2.service" and "journalctl -xeu postgresql@16-main2.service" for details.
была в том, что на конфиги мастера и реплики отличались
max_connections = 100 is a lower setting than on the primary server, where its value was 105.
```
---
```
otus=# SELECT * FROM pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 4994
usesysid         | 10
usename          | postgres
application_name | 16/main2
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2025-07-13 16:57:53.460889+11
backend_xmin     |
state            | streaming
sent_lsn         | 2/C5000060
write_lsn        | 2/C5000060
flush_lsn        | 2/C5000060
replay_lsn       | 2/C5000060
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2025-07-13 16:58:36.909026+11

otus=# SELECT * FROM pg_current_wal_lsn();
 pg_current_wal_lsn
--------------------
 2/C5000060
(1 row)
```

---
#### На реплике
```
otus=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 4993
status                | streaming
receive_start_lsn     | 2/C5000000
receive_start_tli     | 2
written_lsn           | 2/C5000060
flushed_lsn           | 2/C5000060
received_tli          | 2
last_msg_send_time    | 2025-07-13 17:00:07.586954+11
last_msg_receipt_time | 2025-07-13 17:00:07.587036+11
latest_end_lsn        | 2/C5000060
latest_end_time       | 2025-07-13 16:57:53.462657+11
slot_name             |
sender_host           | /var/run/postgresql
sender_port           | 5432
conninfo              | user=postgres passfile=/var/lib/postgresql/.pgpass channel_binding=prefer dbname=replication port=5432 fallback_application_name=16/main2 sslmode=prefer sslcompression=0 sslcertmode=allow sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=prefer krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

otus=# select pg_last_wal_receive_lsn();
 pg_last_wal_receive_lsn
-------------------------
 2/C5000060
(1 row)

otus=# select pg_last_wal_replay_lsn();
 pg_last_wal_replay_lsn
------------------------
 2/C5000060
(1 row)

otus=# select count(*) from student;
 count
-------
   181
(1 row)
```

#### на мастере
```
otus=# insert into student values (201,'ivanov2');
INSERT 0 1
otus=# select count(*) from student;
 count
-------
   182
(1 row)
```

#### на реплике
```
sudo -u postgres psql -p 5433 -d otus

otus=# select count(*) from student;
 count
-------
   182
(1 row)
```

#### Физическая репликация между двумя нодами мастер - слейв настроена.

--- 
#### Каскад 
```
sudo pg_createcluster -d /var/lib/postgresql/16/main3 16 main3
sudo rm -rf /var/lib/postgresql/16/main3
sudo -u postgres pg_basebackup -p 5433 -R -D /var/lib/postgresql/16/main3
sudo pg_ctlcluster 16 main3 start
pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
16  main    5432 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 online postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log
16  main3   5434 online postgres /var/lib/postgresql/16/main3 /var/log/postgresql/postgresql-16-main3.log
```

--- 
#### Проверка
####  мастере 
```
otus=# select count(*) from student;
 count
-------
   182
(1 row)

otus=# insert into student values (203,'ivanov3');
INSERT 0 1
otus=# select count(*) from student;
 count
-------
   183
(1 row)
```

#### на втором каскадном слейве
```
sudo -u postgres psql -p 5434 -d otus

otus=# select count(*) from student;
 count
-------
   183
(1 row)
```

#### Каскад репликации со слейва на третью ноду работает

--- 
#### Логическая репликация таблицы с мастер ноды на четвертую ноду
```
sudo pg_createcluster -d /var/lib/postgresql/16/main4 16 main4
sudo pg_ctlcluster 16 main4 start
```

#### у первого сервера
```
alter system set wal_level = replica;

sudo pg_ctlcluster 16 main restart - такой рестарт у меня перестал пработать мочему-то, что-то справами наверное, вместо этого:
sudo -i -u postgres
/usr/lib/postgresql/16/bin/pg_ctl restart -D /var/lib/postgresql/16/main

otus=# show wal_level;
 wal_level
-----------
 logical
(1 row)

otus=# CREATE PUBLICATION test_pub FOR TABLE student;
```

#### Просмотр созданной публикации
```
otus=# \dRp+
                            Publication test_pub
  Owner   | All tables | Inserts | Updates | Deletes | Truncates | Via root
----------+------------+---------+---------+---------+-----------+----------
 postgres | f          | t       | t       | t       | t         | f
Tables:
    "public.student"
```

#### 4-я нода
```
sudo -u postgres psql -p 5435
postgres=# \dt
Did not find any relations.

postgres=# create database otus;

otus=# create table student as 
select 
  generate_series(1,10) as id,
  md5(random()::text)::char(10) as fio;

otus=# CREATE SUBSCRIPTION otus_sub 
CONNECTION 'host=localhost port=5432 user=postgres password=postgres dbname=otus' 
PUBLICATION test_pub WITH (copy_data = true);
NOTICE:  created replication slot "otus_sub" on publisher
CREATE SUBSCRIPTION

otus=# select count(*) from student;
 count
-------
   193
(1 row)
```

#### 183 записи добавились к имеющимся 10

---
#### на мастере 
```
otus=# insert into student values (204,'ivanov4');
INSERT 0 1
otus=# select count(*) from student;
 count
-------
   184
(1 row)
```

#### на 4-й ноде
```
otus=# select count(*) from student;
 count
-------
   194
(1 row)
```

### Логическая репликация работает.
