# Домашнее задание к уроку №8 - Журналы

*Настройте выполнение контрольной точки раз в 30 секунд.*
Меняем значение параметра в /etc/postgresql/17/main/postgresql.conf

Было:
    #checkpoint_timeout = 5min
Стало:
    checkpoint_timeout = 30s

Рестартуем PostgreSQL для применения новых параметров.

*10 минут c помощью утилиты pgbench подавайте нагрузку.*
    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 600 --protocol=prepared --builtin=simple-update
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: simple update>
    scaling factor: 100
    query mode: prepared
    number of clients: 1
    number of threads: 1
    maximum number of tries: 1
    duration: 600 s
    number of transactions actually processed: 339687
    number of failed transactions: 0 (0.000%)
    latency average = 1.766 ms
    initial connection time = 48.105 ms
    tps = 566.189843 (without initial connection time)

*Измерьте, какой объем журнальных файлов был сгенерирован за это время. Оцените, какой объем приходится в среднем на одну контрольную точку.*
Находим чекпойнты в текстовом логе. Видим, что за время выполнения pgbench прошло 20 чекпойнтов. 
Всего за это время создалось 164 WAL-файла, из них 7 при создании чекпойнтов.
Объем данных можно оценить как 164 * 16 МБ = 2624 МБ, или в среднем 2624 МБ / 20 = 131 МБ за 30-секундный интервал между соседними чекпойнтами.

    2025-07-06 11:52:40.147 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:53:07.033 UTC [1976] LOG:  checkpoint complete: wrote 9570 buffers (7.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.870 s, sync=0.005 s, total=26.886 s; sync files=13, longest=0.002 s, average=0.001 s; distance=75314 kB, estimate=125052 kB; lsn=3/F6B2E9C8, redo lsn=3/EF378220
    2025-07-06 11:53:10.037 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:53:37.016 UTC [1976] LOG:  checkpoint complete: wrote 16776 buffers (12.8%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.968 s, sync=0.003 s, total=26.980 s; sync files=14, longest=0.002 s, average=0.001 s; distance=135816 kB, estimate=135816 kB; lsn=4/C43700, redo lsn=3/F781A580
    2025-07-06 11:53:40.019 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:54:07.105 UTC [1976] LOG:  checkpoint complete: wrote 19847 buffers (15.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=27.061 s, sync=0.005 s, total=27.086 s; sync files=8, longest=0.002 s, average=0.001 s; distance=164504 kB, estimate=164504 kB; lsn=4/903E700, redo lsn=4/18C0648
    2025-07-06 11:54:10.107 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:54:37.045 UTC [1976] LOG:  checkpoint complete: wrote 16346 buffers (12.5%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.925 s, sync=0.004 s, total=26.938 s; sync files=14, longest=0.003 s, average=0.001 s; distance=135108 kB, estimate=161564 kB; lsn=4/1114DC28, redo lsn=4/9CB1890
    2025-07-06 11:54:40.048 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:55:07.160 UTC [1976] LOG:  checkpoint complete: wrote 16103 buffers (12.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=27.014 s, sync=0.057 s, total=27.112 s; sync files=8, longest=0.051 s, average=0.008 s; distance=132130 kB, estimate=158621 kB; lsn=4/19001F78, redo lsn=4/11DBA3C0
    2025-07-06 11:55:10.162 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:55:37.059 UTC [1976] LOG:  checkpoint complete: wrote 15906 buffers (12.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.873 s, sync=0.005 s, total=26.897 s; sync files=17, longest=0.003 s, average=0.001 s; distance=129989 kB, estimate=155758 kB; lsn=4/21710B80, redo lsn=4/19CAB918
    2025-07-06 11:55:40.063 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:56:07.036 UTC [1976] LOG:  checkpoint complete: wrote 16680 buffers (12.7%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.958 s, sync=0.005 s, total=26.974 s; sync files=6, longest=0.003 s, average=0.001 s; distance=137427 kB, estimate=153924 kB; lsn=4/292E8800, redo lsn=4/222E05E8
    2025-07-06 11:56:10.038 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:56:37.230 UTC [1976] LOG:  checkpoint complete: wrote 15537 buffers (11.9%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.968 s, sync=0.004 s, total=27.192 s; sync files=16, longest=0.003 s, average=0.001 s; distance=127585 kB, estimate=151291 kB; lsn=4/30F800A0, redo lsn=4/29F78C80
    2025-07-06 11:56:40.233 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:57:07.059 UTC [1976] LOG:  checkpoint complete: wrote 15644 buffers (11.9%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.798 s, sync=0.007 s, total=26.827 s; sync files=7, longest=0.004 s, average=0.001 s; distance=127895 kB, estimate=148951 kB; lsn=4/38A03298, redo lsn=4/31C5E9D8
    2025-07-06 11:57:10.061 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:57:37.108 UTC [1976] LOG:  checkpoint complete: wrote 15118 buffers (11.5%); 0 WAL file(s) added, 0 removed, 0 recycled; write=27.032 s, sync=0.004 s, total=27.047 s; sync files=15, longest=0.002 s, average=0.001 s; distance=123725 kB, estimate=146428 kB; lsn=4/4033B880, redo lsn=4/39531EB8
    2025-07-06 11:57:40.111 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:58:07.442 UTC [1976] LOG:  checkpoint complete: wrote 15293 buffers (11.7%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.933 s, sync=0.005 s, total=27.331 s; sync files=7, longest=0.002 s, average=0.001 s; distance=125430 kB, estimate=144328 kB; lsn=4/47E9B2B8, redo lsn=4/40FAF7A0
    2025-07-06 11:58:10.445 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:58:37.025 UTC [1976] LOG:  checkpoint complete: wrote 15510 buffers (11.8%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.564 s, sync=0.004 s, total=26.580 s; sync files=12, longest=0.002 s, average=0.001 s; distance=126878 kB, estimate=142583 kB; lsn=4/521060F8, redo lsn=4/48B96FE0
    2025-07-06 11:58:40.028 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:59:07.392 UTC [1976] LOG:  checkpoint complete: wrote 20128 buffers (15.4%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.981 s, sync=0.005 s, total=27.365 s; sync files=7, longest=0.002 s, average=0.001 s; distance=165783 kB, estimate=165783 kB; lsn=4/59C2E8E0, redo lsn=4/52D7CFD8
    2025-07-06 11:59:10.396 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 11:59:37.618 UTC [1976] LOG:  checkpoint complete: wrote 31363 buffers (23.9%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.616 s, sync=0.596 s, total=27.223 s; sync files=7, longest=0.506 s, average=0.086 s; distance=126513 kB, estimate=161856 kB; lsn=4/61AA8560, redo lsn=4/5A9095F0
    2025-07-06 11:59:40.623 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:00:07.553 UTC [1976] LOG:  checkpoint complete: wrote 50530 buffers (38.6%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.056 s, sync=0.089 s, total=26.931 s; sync files=13, longest=0.085 s, average=0.007 s; distance=121844 kB, estimate=157855 kB; lsn=4/67EF7D70, redo lsn=4/62006820
    2025-07-06 12:00:10.556 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:00:37.320 UTC [1976] LOG:  checkpoint complete: wrote 12901 buffers (9.8%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.506 s, sync=0.003 s, total=26.764 s; sync files=13, longest=0.002 s, average=0.001 s; distance=111116 kB, estimate=153181 kB; lsn=4/6FC6BF80, redo lsn=4/68C89910
    2025-07-06 12:00:40.322 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:01:07.043 UTC [1976] LOG:  checkpoint complete: wrote 15828 buffers (12.1%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.710 s, sync=0.003 s, total=26.721 s; sync files=7, longest=0.002 s, average=0.001 s; distance=128202 kB, estimate=150683 kB; lsn=4/78862EF8, redo lsn=4/709BC178
    2025-07-06 12:01:10.046 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:01:37.324 UTC [1976] LOG:  checkpoint complete: wrote 18532 buffers (14.1%); 1 WAL file(s) added, 0 removed, 0 recycled; write=27.014 s, sync=0.003 s, total=27.279 s; sync files=12, longest=0.002 s, average=0.001 s; distance=150161 kB, estimate=150631 kB; lsn=4/80F66E00, redo lsn=4/79C607C8
    2025-07-06 12:01:40.327 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:02:07.348 UTC [1976] LOG:  checkpoint complete: wrote 15895 buffers (12.1%); 1 WAL file(s) added, 0 removed, 0 recycled; write=26.694 s, sync=0.005 s, total=27.021 s; sync files=7, longest=0.002 s, average=0.001 s; distance=131198 kB, estimate=148688 kB; lsn=4/88E69F48, redo lsn=4/81C80040
    2025-07-06 12:02:10.351 UTC [1976] LOG:  checkpoint starting: time
    2025-07-06 12:02:37.107 UTC [1976] LOG:  checkpoint complete: wrote 16147 buffers (12.3%); 0 WAL file(s) added, 0 removed, 0 recycled; write=26.744 s, sync=0.002 s, total=26.757 s; sync files=14, longest=0.002 s, average=0.001 s; distance=129849 kB, estimate=146804 kB; lsn=4/8D446DF8, redo lsn=4/89B4E4F0

*Проверьте данные статистики: все ли контрольные точки выполнялись точно по расписанию. Почему так произошло?*
Да, все контрольные точки выполнялись точно по расписанию.

*Сравните tps в синхронном/асинхронном режиме утилитой pgbench. Объясните полученный результат.*
Выставляем synchronous_commit = off, получаем повышенный tps (638 вместо 566).
Так и должно быть, так как в этом режиме сервер не обязан дожидаться скидывания WAL на диск.

    root@pg-02:~# pgbench -h localhost -U postgres postgres -T 600 --protocol=prepared --builtin=simple-update
    Password:
    pgbench (17.5 (Ubuntu 17.5-1.pgdg24.04+1))
    starting vacuum...end.
    transaction type: <builtin: simple update>
    scaling factor: 100
    query mode: prepared
    number of clients: 1
    number of threads: 1
    maximum number of tries: 1
    duration: 600 s
    number of transactions actually processed: 382867
    number of failed transactions: 0 (0.000%)
    latency average = 1.567 ms
    initial connection time = 47.401 ms
    tps = 638.160466 (without initial connection time)

*Создайте новый кластер с включенной контрольной суммой страниц. Создайте таблицу. Вставьте несколько значений. Выключите кластер. Измените пару байт в таблице. Включите кластер и сделайте выборку из таблицы. Что и почему произошло? как проигнорировать ошибку и продолжить работу?*

Запустили новый кластер:
    postgres=# show data_checksums;
    data_checksums
    ----------------
    on
    (1 row)

Создали таблицу:
    postgres=# create table test2 (a text);
    CREATE TABLE
    postgres=# insert into test2 (a) values ('aaaaaa');
    INSERT 0 1
    postgres=# insert into test2 (a) values ('aaaaaa123213');
    INSERT 0 1
    postgres=# insert into test2 (a) values ('aaaaaa1232143243242343243');
    INSERT 0 1

    postgres=# SELECT pg_relation_filepath('test2');
    pg_relation_filepath
    ----------------------
    base/5/16388
    (1 row)

Выключили сервер:
    postgres@pg-02:/root$ /usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/17/mroff -l /var/log/postgresql/postgresql-17-mroff.log -o "-p 5434" stop
    waiting for server to shut down.... done
    server stopped

Поменяли данные в файле:
    root@pg-02:~# vim /var/lib/postgresql/17/mroff/base/5/16388

Запустили кластер:
    postgres@pg-02:/root$ /usr/lib/postgresql/17/bin/pg_ctl -D /var/lib/postgresql/17/mroff -l /var/log/postgresql/postgresql-17-mroff.log -o "-p 5434" start
    waiting for server to start.... done
    server started

Select не работает:
    postgres=# select * from test2;
    WARNING:  page verification failed, calculated checksum 29005 but expected 2181
    ERROR:  invalid page in block 0 of relation base/5/16388

*Что и почему произошло?*
Таблица не прошла проверку контрольной суммы.

*как проигнорировать ошибку и продолжить работу?*
Выставляем параметр, позволяющий это обойти:
    postgres=# set ignore_checksum_failure = on;
    SET

Обходим:
    postgres=# select * from test2;
    WARNING:  page verification failed, calculated checksum 29005 but expected 2181
                a
    ---------------------------
    addsffesdf2ewfaaaa1232143
    (1 row)

Конец.