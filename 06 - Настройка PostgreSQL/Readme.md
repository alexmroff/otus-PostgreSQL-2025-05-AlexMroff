# Домашнее задание к уроку №6 - Настройка PostgreSQL

## Часть 1. Конфиг

Я буду проводить бенчмарк на виртулке Ubuntu с установленным PostgreSQL 17:

    postgres=# SELECT version();
                                                                version
    -----------------------------------------------------------------------------------------------------------------------------------
    PostgreSQL 17.5 (Ubuntu 17.5-1.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
    (1 row)

Параметры виртуалки: 4 vCPU, 8 GB RAM.

Для получения оптимальных настроек использую https://pgtune.leopard.in.ua/ с выбранным OLTP-приложением.
*Number of connections = 20
Data Storage = HDD Storage*

Получившийся конфиг:

    # DB Version: 17
    # OS Type: linux
    # DB Type: oltp
    # Total Memory (RAM): 8 GB
    # CPUs num: 4
    # Connections num: 20
    # Data Storage: hdd

    max_connections = 20
    shared_buffers = 2GB
    effective_cache_size = 6GB
    maintenance_work_mem = 512MB
    checkpoint_completion_target = 0.9
    wal_buffers = 16MB
    default_statistics_target = 100
    random_page_cost = 4
    effective_io_concurrency = 2
    work_mem = 87381kB
    huge_pages = off
    min_wal_size = 2GB
    max_wal_size = 8GB
    max_worker_processes = 4
    max_parallel_workers_per_gather = 2
    max_parallel_workers = 4
    max_parallel_maintenance_workers = 2

Применяем его и выборочно проверяем настройки. Видим, что настройки применились:

    postgres=# SHOW shared_buffers;
    shared_buffers
    ----------------
    2GB
    (1 row)

    postgres=# SHOW work_mem;
    work_mem
    ----------
    87381kB
    (1 row)

## Часть 2. pgbench

Инициализируем pgbench.
**Сразу делаю оговорку, что тестовый сервер - крайне старенький, HP Proliant Gen 7, причем без SSD-дисков...**

    root@pg-02:~# pgbench --initialize -h localhost -U postgres postgres --scale=100
    Password:
    dropping old tables...
    NOTICE:  table "pgbench_accounts" does not exist, skipping
    NOTICE:  table "pgbench_branches" does not exist, skipping
    NOTICE:  table "pgbench_history" does not exist, skipping
    NOTICE:  table "pgbench_tellers" does not exist, skipping
    creating tables...
    generating data (client-side)...
    vacuuming...
    creating primary keys...
    done in 56.10 s (drop tables 0.03 s, create tables 0.10 s, client-side generate 41.96 s, vacuum 0.63 s, primary keys 13.38 s).

Первый SELECT-тест:

    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 10 --protocol=prepared --builtin=select
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: select only>
    scaling factor: 100
    query mode: prepared
    number of clients: 1
    number of threads: 1
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 28722
    number of failed transactions: 0 (0.000%)
    latency average = 0.347 ms
    initial connection time = 45.353 ms
    tps = 2885.165356 (without initial connection time)

Добавим concurrency:

    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 10 --protocol=prepared --builtin=select --jobs=20 --client=20
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: select only>
    scaling factor: 100
    query mode: prepared
    number of clients: 20
    number of threads: 20
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 197707
    number of failed transactions: 0 (0.000%)
    latency average = 0.989 ms
    initial connection time = 237.351 ms
    tps = 20231.266812 (without initial connection time)

Теперь пробуем не SELECT-, а UPDATE-тест.
Работает жуть как медленно:

    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 10 --protocol=prepared --builtin=simple-update
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: simple update>
    scaling factor: 100
    query mode: prepared
    number of clients: 1
    number of threads: 1
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 5292
    number of failed transactions: 0 (0.000%)
    latency average = 1.881 ms
    initial connection time = 48.304 ms
    tps = 531.683066 (without initial connection time)

С concurrency чуть быстрее:

    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 10 --protocol=prepared --builtin=simple-update --jobs=20 --client=20
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: simple update>
    scaling factor: 100
    query mode: prepared
    number of clients: 20
    number of threads: 20
    maximum number of tries: 1
    duration: 10 s
    number of transactions actually processed: 36783
    number of failed transactions: 0 (0.000%)
    latency average = 5.331 ms
    initial connection time = 212.885 ms
    tps = 3751.476679 (without initial connection time)

Результаты удручающие.
**Сервер НУ ОЧЕНЬ ДРЯХЛЫЙ!**

## Часть 3. sysbench

После установки sysbench инициализируем подопытную БД:

    root@pg-02:~# sysbench --db-driver=pgsql --pgsql-db=postgres   --pgsql-user=postgres --tables=10 --table-size=1000000 --pgsql-password=123456  oltp_read_write prepare
    sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

    Creating table 'sbtest1'...
    Inserting 1000000 records into 'sbtest1'
    Creating a secondary index on 'sbtest1'...
    Creating table 'sbtest2'...
    Inserting 1000000 records into 'sbtest2'
    Creating a secondary index on 'sbtest2'...
    Creating table 'sbtest3'...
    Inserting 1000000 records into 'sbtest3'
    Creating a secondary index on 'sbtest3'...
    Creating table 'sbtest4'...
    Inserting 1000000 records into 'sbtest4'
    Creating a secondary index on 'sbtest4'...
    Creating table 'sbtest5'...
    Inserting 1000000 records into 'sbtest5'
    Creating a secondary index on 'sbtest5'...
    Creating table 'sbtest6'...
    Inserting 1000000 records into 'sbtest6'
    Creating a secondary index on 'sbtest6'...
    Creating table 'sbtest7'...
    Inserting 1000000 records into 'sbtest7'
    Creating a secondary index on 'sbtest7'...
    Creating table 'sbtest8'...
    Inserting 1000000 records into 'sbtest8'
    Creating a secondary index on 'sbtest8'...
    Creating table 'sbtest9'...
    Inserting 1000000 records into 'sbtest9'
    Creating a secondary index on 'sbtest9'...
    Creating table 'sbtest10'...
    Inserting 1000000 records into 'sbtest10'
    Creating a secondary index on 'sbtest10'...

Выполянем тест:

    root@pg-02:~# sysbench --db-driver=pgsql --pgsql-db=postgres --pgsql-user=postgres --pgsql-password=123456 --threads=4   --time=60 oltp_read_write run
    sysbench 1.0.20 (using system LuaJIT 2.1.0-beta3)

    Running the test with following options:
    Number of threads: 4
    Initializing random number generator from current time


    Initializing worker threads...

    Threads started!

    SQL statistics:
        queries performed:
            read:                            666974
            write:                           190083
            other:                           95525
            total:                           952582
        transactions:                        47524  (791.95 per sec.)
        queries:                             952582 (15874.06 per sec.)
        ignored errors:                      117    (1.95 per sec.)
        reconnects:                          0      (0.00 per sec.)

    General statistics:
        total time:                          60.0067s
        total number of events:              47524

    Latency (ms):
            min:                                    4.15
            avg:                                    5.05
            max:                                 1015.55
            95th percentile:                        5.77
            sum:                               239926.87

    Threads fairness:
        events (avg/stddev):           11881.0000/74.94
        execution time (avg/stddev):   59.9817/0.00

**NOTE:** как интерпретировать результаты sysbench - я не очень понимаю.
 Прошу проверяющего помочь. 
