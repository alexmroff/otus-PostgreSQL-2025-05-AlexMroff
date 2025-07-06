# Домашнее задание к уроку №7 - MVCC, vacuum и autovacuum

## Часть 1. pgbench

Работаю на PostgreSQL 17 на стааааареньком сервере без SSD.

Показатель TPS очень низкий:

    root@pg-02:~# pgbench -c8 -P 6 -T 60 -U postgres postgres
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    progress: 6.0 s, 1038.5 tps, lat 7.606 ms stddev 4.282, 0 failed
    progress: 12.0 s, 1043.5 tps, lat 7.657 ms stddev 4.384, 0 failed
    progress: 18.0 s, 1046.2 tps, lat 7.640 ms stddev 4.378, 0 failed
    progress: 24.0 s, 1032.7 tps, lat 7.735 ms stddev 4.558, 0 failed
    progress: 30.0 s, 1004.8 tps, lat 7.954 ms stddev 5.962, 0 failed
    progress: 36.0 s, 1289.0 tps, lat 6.201 ms stddev 3.975, 0 failed
    progress: 42.0 s, 1725.8 tps, lat 4.629 ms stddev 3.253, 0 failed
    progress: 48.0 s, 1090.5 tps, lat 7.324 ms stddev 4.393, 0 failed
    progress: 54.0 s, 1049.7 tps, lat 7.612 ms stddev 4.446, 0 failed
    progress: 60.0 s, 1018.1 tps, lat 7.846 ms stddev 5.835, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 8
    number of threads: 1
    maximum number of tries: 1
    duration: 60 s
    number of transactions actually processed: 68040
    number of failed transactions: 0 (0.000%)
    latency average = 7.039 ms
    latency stddev = 4.659 ms
    initial connection time = 61.226 ms
    tps = 1134.823844 (without initial connection time)

После применения настроек он стал еще ниже:

    root@pg-02:~# pgbench -c8 -P 6 -T 60 -U postgres postgres
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    progress: 6.0 s, 993.2 tps, lat 7.950 ms stddev 4.454, 0 failed
    progress: 12.0 s, 1026.8 tps, lat 7.783 ms stddev 4.431, 0 failed
    progress: 18.0 s, 1019.5 tps, lat 7.833 ms stddev 4.548, 0 failed
    progress: 24.0 s, 1016.3 tps, lat 7.866 ms stddev 4.592, 0 failed
    progress: 30.0 s, 1013.7 tps, lat 7.874 ms stddev 4.431, 0 failed
    progress: 36.0 s, 1012.3 tps, lat 7.897 ms stddev 4.532, 0 failed
    progress: 42.0 s, 1030.8 tps, lat 7.752 ms stddev 4.450, 0 failed
    progress: 48.0 s, 1020.3 tps, lat 7.823 ms stddev 4.465, 0 failed
    progress: 54.0 s, 1015.8 tps, lat 7.870 ms stddev 4.452, 0 failed
    progress: 60.0 s, 1016.5 tps, lat 7.862 ms stddev 4.386, 0 failed
    transaction type: <builtin: TPC-B (sort of)>
    scaling factor: 1
    query mode: simple
    number of clients: 8
    number of threads: 1
    maximum number of tries: 1
    duration: 60 s
    number of transactions actually processed: 61000
    number of failed transactions: 0 (0.000%)
    latency average = 7.851 ms
    latency stddev = 4.475 ms
    initial connection time = 64.663 ms
    tps = 1017.454957 (without initial connection time)

Вывод можно сделать только один: нельзя применять к своему конкретному серверу рандомные настройки, даже если они выданы преподавателем!

## Часть 2. Autovacuum.

Создали таблицу:

    postgres=# CREATE TABLE test_av (t1 text);
    CREATE TABLE

Инсертим мильон строк:

    postgres=# INSERT INTO test_av
    postgres-# SELECT substr(md5(random()::text), 1, 10)
    postgres-# FROM generate_series(1, 1000000);
    INSERT 0 1000000

Размер таблицы = 42 МБ:

    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    42 MB
    (1 row)

Все записи живы:

    postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
    postgres-# FROM pg_stat_user_tables WHERE relname = 'test_av';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
    ---------+------------+------------+--------+-------------------------------
    test_av |    1000000 |          0 |      0 | 2025-06-28 09:41:05.272482+00
    (1 row)

Делаем 5 апдейтов:

    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '33';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '44';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '55';
    UPDATE 1000000

Файл вырос в размере:

    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    276 MB
    (1 row)

Однако автовакуум вычистил таблицу почти мгновенно...

    postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
    FROM pg_stat_user_tables WHERE relname = 'test_av';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
    ---------+------------+------------+--------+-------------------------------
    test_av |    1000000 |          0 |      0 | 2025-06-28 09:50:06.489442+00
    (1 row)

Делаем еще 5 апдейтов и убеждаемся, что файл растет:

    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '33';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '44';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '55';
    UPDATE 1000000
    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    329 MB
    (1 row)

Отключаем автовакуум:

    postgres=# ALTER TABLE test_av SET (autovacuum_enabled = off);
    ALTER TABLE

Делаем еще 10 апдейтов:

    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '11';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000
    postgres=# UPDATE test_av SET t1 = t1 || '22';
    UPDATE 1000000

Таблица растет:

    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    771 MB
    (1 row)

Автовакуум не приходит, в таблице огромное количество мертвых записей:

    postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
    FROM pg_stat_user_tables WHERE relname = 'test_av';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
    ---------+------------+------------+--------+-------------------------------
    test_av |    1000000 |    9997321 |    999 | 2025-06-28 09:54:07.215117+00
    (1 row)

Система работает как и должна.
Включаем автовакуум обратно:

    postgres=# ALTER TABLE test_av SET (autovacuum_enabled = on);
    ALTER TABLE

Ждем несколько секунд... автовакуум сработал:

    postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum
    FROM pg_stat_user_tables WHERE relname = 'test_av';
    relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
    ---------+------------+------------+--------+-------------------------------
    test_av |    1000000 |          0 |      0 | 2025-06-28 10:00:15.444327+00
    (1 row)

А размер файла не уменьшился:

    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    771 MB
    (1 row)

Фиксируем путь к файлу с таблицей:

    postgres=# SELECT pg_relation_filepath('test_av');
    pg_relation_filepath
    ----------------------
    base/5/16428
    (1 row)

Запускаем VACUUM FULL:

    postgres=# VACUUM FULL test_av;
    VACUUM

Размер таблицы уменьшился:

    postgres=# SELECT pg_size_pretty(pg_total_relation_size('test_av'));
    pg_size_pretty
    ----------------
    81 MB
    (1 row)

А сама таблица переехала в другой файл:

    postgres=# SELECT pg_relation_filepath('test_av');
    pg_relation_filepath
    ----------------------
    base/5/16437
    (1 row)

Задание выполнено.

## Часть 3. Анонимная процедура.

Анонимная процедура написана и работает, но сожрала всё место на моем маленькм тестовом диске...

    postgres=# DO
    $do$
    BEGIN
    FOR i IN 1..10 LOOP
    RAISE NOTICE 'Iteration %', i;
    UPDATE test_av SET t1 = t1 || 'a';
    END LOOP;
    END
    $do$;
    NOTICE:  Iteration 1
    NOTICE:  Iteration 2
    NOTICE:  Iteration 3
    NOTICE:  Iteration 4
    NOTICE:  Iteration 5
    NOTICE:  Iteration 6
    NOTICE:  Iteration 7
    NOTICE:  Iteration 8
    NOTICE:  Iteration 9
    PANIC:  could not write to file "pg_wal/xlogtemp.8280": No space left on device
    CONTEXT:  SQL statement "UPDATE test_av SET t1 = t1 || 'a'"
    PL/pgSQL function inline_code_block line 5 at SQL statement
    server closed the connection unexpectedly
            This probably means the server terminated abnormally
            before or while processing the request.
    The connection to the server was lost. Attempting reset: Failed.
    The connection to the server was lost. Attempting reset: Failed.
    !?>

Высвобождение дискового пространства будет выполнено за рамками данной домашней работы )))