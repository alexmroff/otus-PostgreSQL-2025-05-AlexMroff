# Домашнее задание к уроку №5 - Логический уровень PostgreSQL

Работаю с версией 17, не 14:

    postgres=# SELECT version();
                                                                version
    -----------------------------------------------------------------------------------------------------------------------------------
    PostgreSQL 17.5 (Ubuntu 17.5-1.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
    (1 row)

Создаю БД:

    postgres=# CREATE DATABASE testdb;
    CREATE DATABASE

Заходим в нее:

    postgres=# \c testdb
    You are now connected to database "testdb" as user "postgres".
    testdb=#

Создаем схему:

    testdb=# CREATE SCHEMA testnm;
    CREATE SCHEMA

Создаем таблицу:

    testdb=# CREATE TABLE t1 (c1 integer);
    CREATE TABLE

Инсертим в нее:

    testdb=# INSERT INTO t1 (c1) VALUES (1);
    INSERT 0 1

Создаем роль:

    testdb=# CREATE ROLE readonly WITH LOGIN PASSWORD '123456';
    CREATE ROLE

Разрешаем коннект к БД:

    testdb=# GRANT CONNECT ON DATABASE testdb TO readonly;
    GRANT

Разрешаем использование схемы:

    testdb=# GRANT USAGE ON SCHEMA testnm TO readonly;
    GRANT

Разрешаем чтение из всех таблиц:

    testdb=# GRANT SELECT ON ALL TABLES IN SCHEMA testnm TO readonly;
    GRANT

Создаем пользователя:

    testdb=# CREATE USER testread WITH PASSWORD 'Test123';
    CREATE ROLE

Прикручиваем ему роль:

    testdb=# GRANT readonly TO testread;
    GRANT ROLE

Перелогиниваемся в базу, проверяем, кто мы, и пробуем SELECT:

    testdb=> SELECT session_user, current_user;
    session_user | current_user
    --------------+--------------
    testread     | testread
    (1 row)

    testdb=> SELECT * FROM t1;
    ERROR:  permission denied for table t1

Доступа нет. Причина - таблица создана в схеме public, а не testnm:

    testdb=> \dt
            List of relations
    Schema | Name | Type  |  Owner
    --------+------+-------+----------
    public | t1   | table | postgres
    (1 row)

Удаляем таблицу от имени postgres:

    testdb=# DROP TABLE t1;
    DROP TABLE

Создаем таблицу в нужной схеме:

    testdb=# CREATE TABLE testnm.t1 (c1 integer);
    CREATE TABLE

Инсертим:

    testdb=# INSERT INTO testnm.t1 (c1) VALUES (1);
    INSERT 0 1

Перелогиниваемся:

    testdb=# \c testdb testread
    You are now connected to database "testdb" as user "testread".

Селект снова не срабатывает:

    testdb=> SELECT * FROM testnm.t1;
    ERROR:  permission denied for table t1

Для поиска причин пришлось подсмотреть в шпаргалку.
Выходит, что права применились только к тем таблицам, которые были в БД на момент выдачи прав, а таблица testnm.t1 была создана позже.
Любопытно...

Меняем юзера:

    testdb=> \c testdb postgres;
    You are now connected to database "testdb" as user "postgres".

Меняем дефолтные привилегии:

    testdb=# ALTER DEFAULT PRIVILEGES IN SCHEMA testnm GRANT SELECT ON TABLES TO readonly;
    ALTER DEFAULT PRIVILEGES

Пробуем заново:

    testdb=# \c testdb testread;
    You are now connected to database "testdb" as user "testread".
    testdb=> SELECT * FROM testnm.t1;
    ERROR:  permission denied for table t1

Тут как раз причина ошибки понятна, дефолтные привилегии будут применяться к новым объектам, а не к существующим.
Выдаем права явно:

    testdb=> \c testdb postgres;
    You are now connected to database "testdb" as user "postgres".
    testdb=# GRANT SELECT ON testnm.t1 TO readonly;
    GRANT
    testdb=# \c testdb testread;
    You are now connected to database "testdb" as user "testread".
    testdb=> SELECT * FROM testnm.t1;
    c1
    ----
    1
    (1 row)

Пытаемся создать таблицу:

    testdb=> CREATE TABLE t2 (c1 integer);
    ERROR:  permission denied for schema public
    LINE 1: CREATE TABLE t2 (c1 integer);

Выходит, что в PostgreSQL 17 для схемы public по умолчанию нет прав на CREATE;
Тем не менее отберем все права согласно шпаргалки, пусть одна из команд в версии 17 выходит ненужной:

    testdb=> \c testdb postgres;
    You are now connected to database "testdb" as user "postgres".
    testdb=# REVOKE CREATE ON SCHEMA public FROM public;
    REVOKE
    testdb=# REVOKE ALL on DATABASE testdb FROM public;
    REVOKE

Выполнять пункт 41 в данном случае смысла нет.

**А значит ДЗ выполнено!**