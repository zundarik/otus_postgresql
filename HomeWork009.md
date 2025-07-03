### Описание/Пошаговая инструкция выполнения домашнего задания:
- Создать бекапы с помощью pg_dump, pg_dumpall и pg_basebackup сравнить скорость создания и возможности.
- Настроить копирование WAL файлов.
- Восстановить базу на другой машине PostgreSQL на заданное время, используя ранее созданные бекапы и WAL файлы.

---
```
postgres=# \c otus  
otus=# \dt  
          List of relations  
 Schema |   Name   | Type  |  Owner   
--------+----------+-------+----------  
 public | accounts | table | postgres  
 public | test2    | table | postgres  
(2 rows)  
```

---  
#### Создание таблицы  
```
otus=# create table student as   
select  
  generate_series(1,10) as id,  
  md5(random()::text)::char(10) as fio;  
```
```
otus=# \dt+  
                                       List of relations  
 Schema |   Name   | Type  |  Owner   | Persistence | Access method |    Size    | Description  
--------+----------+-------+----------+-------------+---------------+------------+-------------  
 public | accounts | table | postgres | permanent   | heap          | 16 kB      |  
 public | student  | table | postgres | permanent   | heap          | 8192 bytes |  
 public | test2    | table | postgres | permanent   | heap          | 8192 bytes |  
(3 rows)  
```
```
otus=# \d student  
                 Table "public.student"  
 Column |     Type      | Collation | Nullable | Default  
--------+---------------+-----------+----------+---------  
 id     | integer       |           |          |  
 fio    | character(10) |           |          |  
```
```
otus=# select count(*) from student;  
 count  
-------  
    10  
(1 row)  
```

---  
#### Копирование данных таблицы в файл  
```
otus=# \copy student to '/mnt/w/postgres/otus/student.sql';  
COPY 10  
```

--- pg_dump --- Копия базы otus  
```
sudo -u postgres  

pg_dump -d otus -C -U postgres -p 5432 > /mnt/w/postgres/otus/otus_arh.sql  
 
psql -U postgres -p 5432 otus < /mnt/w/postgres/otus/otus_arh.sql  

... несколько раз  

otus=# select count(*) from student;  
 count  
-------  
    60  
(1 row)  
```

---  
#### Копия базы в архив  
```
pg_dump -d otus --create -U postgres -Fc -p 5432 > /mnt/w/postgres/otus/otus_arh2.gz  
pg_restore -d otus -U postgres -p 5432 /mnt/w/postgres/otus/otus_arh2.gz  
```
```
pg_restore: error: could not execute query: ERROR:  relation "student" already exists  
Command was: CREATE TABLE public.student (  
    id integer,  
    fio character(10)  
);  
```
```
otus=# select count(*) from student;  
 count  
-------  
   180  
(1 row)  
```

---
#### pg_dumpall		Копия кластера		PostgreSQL database cluster dump  
```
pg_dumpall -U postgres -p 5432 > /mnt/w/postgres/otus/bkup_all.sql  
psql -U postgres -p 5432 < /mnt/w/postgres/otus/bkup_all.sql  
```
Заняло времени почти пять минут, файл с архивом размером 1.2 Гб

---
#### Физический бекап
```
otus=# show wal_level;
 wal_level
-----------
 replica
(1 row)
```
```
pg_createcluster -d /var/lib/postgresql/16/main2 16 main2
rm -rf /var/lib/postgresql/16/main2
```
---

#### Копия кластера
```
sudo -u postgres pg_basebackup -p 5432 -D /var/lib/postgresql/16/main2

pg_ctlcluster 16 main2 start

pg_lsclusters
Ver Cluster Port Status Owner    Data directory               Log file
16  main    5432 online postgres /var/lib/postgresql/16/main  /var/log/postgresql/postgresql-16-main.log
16  main2   5433 online postgres /var/lib/postgresql/16/main2 /var/log/postgresql/postgresql-16-main2.log

pg_dropcluster 16 main2 --stop		УДАЛЕНИЕ кластера
```

---
####
```
otus=# SELECT name, setting FROM pg_settings WHERE name IN ('archive_mode','archive_command','archive_timeout');
      name       |  setting
-----------------+------------
 archive_command | (disabled)
 archive_mode    | off
 archive_timeout | 0
(3 rows)
```
```
sudo mkdir /archive_wal
sudo chown -R postgres:postgres /archive_wal
```
```
ALTER SYSTEM SET archive_mode = 'on';
ALTER SYSTEM SET archive_command = 'test ! -f /archive_wal/%f && cp %p /archive_wal/%f';
```
```
sudo systemctl restart 

sudo mkdir /full_backup
sudo chown -R postgres:postgres /full_backup
```
```
sudo -u postgres pg_basebackup -p 5432 -v -D /full_backup
pg_basebackup: initiating base backup, waiting for checkpoint to complete
pg_basebackup: checkpoint completed
pg_basebackup: write-ahead log start point: 2/A9000028 on timeline 1
pg_basebackup: starting background WAL receiver
pg_basebackup: created temporary replication slot "pg_basebackup_4746"
pg_basebackup: write-ahead log end point: 2/A9000138
pg_basebackup: waiting for background process to finish streaming ...
pg_basebackup: syncing data to disk ...
pg_basebackup: renaming backup_manifest.tmp to backup_manifest
pg_basebackup: base backup completed
```

---
```
create table test (c1 text);
insert into test values ('Проверка восстановления с использованием WAL');

otus=# select now();
              now
-------------------------------
 2025-07-03 12:46:33.318921+11
(1 row)

update student set fio = 'Иванова';
select * from student;
```

---
#### Восстановления на определенный момент времени
```
sudo systemctl stop postgresql

sudo mkdir /old_data
sudo chown -R postgres:postgres /old_data
```

#### Сохранение текущего состояния данных Postgres
```
sudo mv /var/lib/postgresql/16/main /old_data			!!! перенос, не копирование

sudo mkdir /var/lib/postgresql/16/main
sudo chown -R postgres:postgres /var/lib/postgresql/16/main

sudo cp -a /full_backup/. /var/lib/postgresql/16/main

sudo rm -rf /var/lib/postgresql/16/main/pg_wal/*
sudo cp -a /old_data/main/pg_wal/. /var/lib/postgresql/16/main/pg_wal
```
```
/etc/postgresql/16/main/postgresql.conf
restore_command = 'cp /archive_wal/%f "%p"'
recovery_target_time = '2025-07-03 12:46:33.318921+11'
```
```
sudo touch /var/lib/postgresql/16/main/recovery.signal
sudo chown -R postgres:postgres /var/lib/postgresql/16/main/
sudo chmod -R 750 /var/lib/postgresql/16/main/
```
```
sudo systemctl start postgresql
```
```
otus=# select * from test;
                      c1
----------------------------------------------
 Проверка восстановления с использованием WAL
(1 row)

otus=# select * from student;
id |    fio
----+------------
  1 | 0a6c457efb
  2 | c01a7faa33
  3 | 90c381ba88
  4 | 0c9aacf4df
  5 | 06c5505e45
...
```
```
select pg_wal_replay_resume();
```

