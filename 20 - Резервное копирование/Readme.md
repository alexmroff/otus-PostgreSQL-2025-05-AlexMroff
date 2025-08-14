# Домашнее задание к уроку №20 - Резервное копирование и восстановление

Используем кластер PostgreSQL 17

    root@pg-02:~# pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    16  main    5433 online postgres /var/lib/postgresql/16/main /var/log/postgresql/postgresql-16-main.log
    17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

Создаем БД, схему, таблицы:

    postgres=# CREATE DATABASE kosmos;
    CREATE DATABASE
    postgres=# \c kosmos;
    You are now connected to database "kosmos" as user "postgres".
    kosmos=# CREATE SCHEMA mars;
    CREATE SCHEMA
    kosmos=# CREATE TABLE mars.m1 AS
    SELECT
    generate_series(1,100) as id,
    md5(random()::text)::char(10) as fio;
    SELECT 100
    kosmos=# SELECT count (*) FROM mars.m1;
    count
    -------
    100
    (1 row)
    kosmos=# CREATE TABLE mars.m2 AS SELECT * FROM mars.m1 LIMIT 0;
    SELECT 0
    kosmos=# SELECT COUNT (*) FROM mars.m2;
    count
    -------
        0
    (1 row)

Создаем папку и выдаем права:

    root@pg-02:~# pwd
    /root
    root@pg-02:~# mkdir copy
    root@pg-02:~# chown postgres copy
    root@pg-02:~# ll copy
    total 8
    drwxr-xr-x 2 postgres root 4096 Aug 14 19:47 ./
    drwx------ 5 root     root 4096 Aug 14 19:47 ../

Копируем первую таблицу в файл, а потом файл загружаем во вторую таблицу:

    kosmos=# \copy mars.m1 to /root/copy/m1.sql
    COPY 100
    kosmos=# \copy mars.m2 from /root/copy/m1.sql
    COPY 100
    kosmos=# select count (*) from mars.m2;
    count
    -------
    100
    (1 row)

Сделаем бэкап обеих таблиц через pg_dump:

    root@pg-02:~# pg_dump -U postgres -d kosmos -t mars.m1 -t mars.m2 -Fc > /root/copy/backup.gz

Удалим вторую таблицу:

    kosmos=# DROP table mars.m2;
    DROP TABLE

Восстановим из сделанного бэкапа токмо вторую таблицу:

    root@pg-02:~# pg_restore -U postgres -d kosmos -t m2 -n mars /root/copy/backup.gz
    root@pg-02:~# psql -U postgres -d kosmos
    psql (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    Type "help" for help.

    kosmos=# SELECT count (*) FROM mars.m2;
    count
    -------
    100
    (1 row)

Задача выполнена!