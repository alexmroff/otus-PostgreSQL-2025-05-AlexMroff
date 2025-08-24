# Домашнее задание к уроку №21 - Репликация

Инфраструктура для выполнения ТЗ: 3 виртуальные машины Ubuntu 24.04.3: pg-01, pg-02, pg-03

# Установка PostgreSQL

Устанавливаем PostgreSQL на всех трех ВМ:

    sudo apt install -y postgresql-common
    sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh
    sudo apt update

Применяем best practices в /etc/postgresql/17/main/postgresql.conf:

    max_connections = 25
    shared_buffers = 2GB
    effective_cache_size = 6GB
    maintenance_work_mem = 512MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 4
    effective_io_concurrency = 2
    work_mem = 72315kB
    huge_pages = off
    min_wal_size = 2GB
    max_wal_size = 8GB
    max_worker_processes = 4
    max_parallel_workers_per_gather = 2
    max_parallel_workers = 4
    max_parallel_maintenance_workers = 2
    wal_level = logical
    listen_addresses = '*'

Получаем 3 одинаковых сервера PostgreSQL:

    mroff@pg-01:~$ sudo pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

    mroff@pg-02:~$ sudo pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

    mroff@pg-03:~$ sudo pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

# Настройка pg_hba.conf

Настраивам доступ для репликаций с нужных IP адресов на первых двух серверах.

Было:

    host    replication     all             127.0.0.1/32            scram-sha-256

Стало:

    host    replication     all             192.168.2.0/24            scram-sha-256

Задаем на всех трех серверах пароль для пользователя postgres и разрешаем ему сетевой доступ через pg_hba.conf:

    mroff@pg-01:~$ sudo -u postgres psql
    psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
    Type "help" for help.

    postgres=# \password postgres
    Enter new password for user "postgres":
    Enter it again:

pg_hba.conf:

    host    all             postgres        192.168.155.0/24        scram-sha-256

Перезагружаем конфигурацию и проверяем коннект из pgAdmin:

    postgres=# SELECT pg_reload_conf();
    pg_reload_conf
    ----------------
    t
    (1 row)

# Подготовка серверов к репликации

Создаем на каждом сервере БД, схему, 2 таблицы.

    postgres=# CREATE DATABASE REPLTEST;
    CREATE DATABASE
    postgres=# \c repltest;
    You are now connected to database "repltest" as user "postgres".
    repltest=# CREATE SCHEMA replicas;
    CREATE SCHEMA
    repltest=# CREATE TABLE replicas.test (id integer);
    CREATE TABLE
    repltest=# CREATE TABLE replicas.test2 (ppp text);
    CREATE TABLE

# Учетные записи для репликации

На первых двух серверах создаю учетку:

    CREATE ROLE repluser WITH
        LOGIN
        NOSUPERUSER
        NOCREATEDB
        NOCREATEROLE
        INHERIT
        REPLICATION
        NOBYPASSRLS
        CONNECTION LIMIT -1
        PASSWORD 'xxxxxx';

И выдаю ей права - на первом сервере на таблицу test, а на втором - на таблицу test2.

pg-01:

    GRANT USAGE ON SCHEMA replicas TO repluser;
    GRANT SELECT ON TABLE replicas.test TO repluser;

pg-02:

    GRANT USAGE ON SCHEMA replicas TO repluser;
    GRANT SELECT ON TABLE replicas.test2 TO repluser;

# Публикации и подписки

На первом и втором серверах создаем публикации.

pg-01:

    CREATE PUBLICATION pub1
        FOR TABLE replicas.test
        WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);

pg-02:

    CREATE PUBLICATION pub2
    FOR TABLE replicas.test2
    WITH (publish = 'insert, update, delete, truncate', publish_via_partition_root = false);

На первом и втором сервере создаем по одной подписке, а на третьем - две.

pg-01:

    CREATE SUBSCRIPTION sub1
        CONNECTION 'host=192.168.2.83 port=5432 user=repluser dbname=repltest connect_timeout=10 sslmode=prefer'
        PUBLICATION pub2
        WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');

pg-02:

    CREATE SUBSCRIPTION sub02
        CONNECTION 'host=192.168.2.69 port=5432 user=repluser dbname=repltest connect_timeout=10 sslmode=prefer'
        PUBLICATION pub1
        WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');

pg-03:

    CREATE SUBSCRIPTION sub03
        CONNECTION 'host=192.168.2.69 port=5432 user=repluser dbname=repltest connect_timeout=10 sslmode=prefer'
        PUBLICATION pub1
        WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');

    CREATE SUBSCRIPTION sub04
        CONNECTION 'host=192.168.2.83 port=5432 user=repluser dbname=repltest connect_timeout=10 sslmode=prefer'
        PUBLICATION pub2
        WITH (connect = true, enabled = true, copy_data = true, create_slot = true, synchronous_commit = 'off', binary = false, streaming = 'False', two_phase = false, disable_on_error = false, run_as_owner = false, password_required = true, origin = 'any');    

# Инсертим и проверяем

## БД на pg-01

До:

pg-01:

    mroff@pg-01:~$ hostname
    pg-01
    mroff@pg-01:~$ sudo -u postgres psql
    psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
    Type "help" for help.

    postgres=# \c repltest
    You are now connected to database "repltest" as user "postgres".
    repltest=# select * from replicas.test;
    id
    ----
    (0 rows)

pg-02:

    mroff@pg-02:~$ hostname
    pg-02
    mroff@pg-02:~$ sudo -u postgres psql
    psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
    Type "help" for help.

    postgres=# \c repltest
    You are now connected to database "repltest" as user "postgres".
    repltest=# select * from replicas.test;
    id
    ----
    (0 rows)

pg-03:

    mroff@pg-03:~$ hostname
    pg-03
    mroff@pg-03:~$ sudo -u postgres psql
    psql (17.6 (Ubuntu 17.6-1.pgdg24.04+1))
    Type "help" for help.

    postgres=# \c repltest
    You are now connected to database "repltest" as user "postgres".
    repltest=# select * from replicas.test;
    id
    ----
    (0 rows)

Деалем insert на pg-01:

    repltest=# insert into replicas.test (id) values (1), (2), (3);
    INSERT 0 3
    repltest=# select * from replicas.test;
    id
    ----
    1
    2
    3
    (3 rows)

pg-02:

    repltest=# select * from replicas.test;
    id
    ----
    1
    2
    3
    (3 rows)

pg-03:

    repltest=# select * from replicas.test;
    id
    ----
    1
    2
    3
    (3 rows)

**Логическая репликация работает!**

## БД на pg-02

pg-02:

    repltest=# INSERT INTO replicas.test2 (ppp) VALUES ('aaaaaaaaa'), ('0000000000000000'), ('xxxxxxxxxxxxxxxxxxxxxxxxxxxx');
    INSERT 0 3
    repltest=# select * from replicas.test2;
                ppp
    ------------------------------
    aaaaaaaaa
    0000000000000000
    xxxxxxxxxxxxxxxxxxxxxxxxxxxx
    (3 rows)

pg-01:

    repltest=# select * from replicas.test2;
                ppp
    ------------------------------
    aaaaaaaaa
    0000000000000000
    xxxxxxxxxxxxxxxxxxxxxxxxxxxx
    (3 rows)

pg-03:

    repltest=# select * from replicas.test2;
                ppp
    ------------------------------
    aaaaaaaaa
    0000000000000000
    xxxxxxxxxxxxxxxxxxxxxxxxxxxx
    (3 rows)

**Логическая репликация работает!**
**Домашнее задание выполнено!!**
