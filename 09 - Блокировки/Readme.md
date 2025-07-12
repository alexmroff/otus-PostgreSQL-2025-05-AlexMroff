# Домашнее задание к уроку №8 - Блокировки

## Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

Выставляем параметры в postgresql.conf и рестартуем кластер:

    deadlock_timeout = 200ms
    log_lock_waits = on 

    root@pg-02:~# pg_ctlcluster 17 main restart

## Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

Создаем таблицу:
    postgres=# create table lockstest (id int, speech text);
    CREATE TABLE
    postgres=# insert into lockstest (id, speech) values (1, 'aaaa');
    INSERT 0 1
    postgres=# insert into lockstest (id, speech) values (2, 'bbbb');
    INSERT 0 1
    postgres=# insert into lockstest (id, speech) values (3, 'cccc');
    INSERT 0 1

Запоминаем id БД и таблицы:
    postgres=# SELECT pg_relation_filepath('lockstest');
    pg_relation_filepath
    ----------------------
    base/5/24938
    (1 row)

Запускаем 3 сессии psql, отключаем autommit и запускаем транзакцию в каждой:

    postgres=# \set AUTOCOMMIT off
    postgres=# begin;
    BEGIN
    postgres=*#

Запускаем три апдейта одной и той же строки:

    postgres=*# UPDATE lockstest SET speech = 'ddddd' WHERE id = 1;
    UPDATE 1

    postgres=*# UPDATE lockstest SET speech = 'eeeee' WHERE id = 1;

    postgres=*# UPDATE lockstest SET speech = 'fffff' WHERE id = 1;

Видно, что выполнился только первый апдейт.

Событие блокировки залогировалось:
    root@pg-02:~# grep ShareLock /var/log/postgresql/postgresql-17-main.log
    2025-07-07 05:13:44.021 UTC [13277] postgres@postgres LOG:  process 13277 still waiting for ShareLock on transaction 1631258 after 200.181 ms

Смотрим pg_locks:

    postgres=# select * from pg_locks where database = 5 and relation = 24938;
    locktype | database | relation | page | tuple | virtualxid | transactionid | classid | objid | objsubid | virtualtransaction |  pid  |       mode       | granted | fastpath |           waitstart        
    ----------+----------+----------+------+-------+------------+---------------+---------+-------+----------+--------------------+-------+------------------+---------+----------+-------------------------------
    relation |        5 |    24938 |      |       |            |               |         |       |          | 3/8                | 66550 | RowExclusiveLock | t       | t        |
    relation |        5 |    24938 |      |       |            |               |         |       |          | 4/3                | 66573 | RowExclusiveLock | t       | t        |
    relation |        5 |    24938 |      |       |            |               |         |       |          | 5/2                | 66575 | RowExclusiveLock | t       | t        |
    tuple    |        5 |    24938 |    0 |     1 |            |               |         |       |          | 5/2                | 66575 | ExclusiveLock    | f       | f        | 2025-07-12 08:48:35.902868+00
    tuple    |        5 |    24938 |    0 |     1 |            |               |         |       |          | 4/3                | 66573 | ExclusiveLock    | t       | f        |
    (5 rows)

Видим 5 блокировок на таблице.

3 из них имеют locktype: relation и mode: RowExclusiveLock.
Эти блокировки вызваны запуском команды UPDATE.

2 из них имеют locktype: tuple и mode: ExclusiveLock
Они указыывают какой кортеж в каком отношении заблокирован.

## Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Да, такие ситуации логируются:

    2025-07-12 08:47:54.162 UTC [66573] postgres@postgres STATEMENT:  postgres=*# UPDATE lockstest SET speech = 'eeeee' WHERE id = 1;
    2025-07-12 08:48:26.143 UTC [66573] postgres@postgres LOG:  process 66573 still waiting for ShareLock on transaction 1631261 after 200.135 ms
    2025-07-12 08:48:26.143 UTC [66573] postgres@postgres DETAIL:  Process holding the lock: 66550. Wait queue: 66573.
    2025-07-12 08:48:26.143 UTC [66573] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "lockstest"
    2025-07-12 08:48:26.143 UTC [66573] postgres@postgres STATEMENT:  UPDATE lockstest SET speech = 'eeeee' WHERE id = 1;
    2025-07-12 08:48:36.103 UTC [66575] postgres@postgres LOG:  process 66575 still waiting for ExclusiveLock on tuple (0,1) of relation 24938 of database 5 after 200.251 ms
    2025-07-12 08:48:36.103 UTC [66575] postgres@postgres DETAIL:  Process holding the lock: 66573. Wait queue: 66575.
    2025-07-12 08:48:36.103 UTC [66575] postgres@postgres STATEMENT:  UPDATE lockstest SET speech = 'fffff' WHERE id = 1;

## Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

Получается, что могут, поскольку транзакции при выполнении UPDATE накладывают Exclusive Lock на строки, то если они попытаются одновременно одну и ту же строку, то попадут в deadlock.

