# Домашнее задание к уроку №4 - Физический уровень PostgreSQL

## Часть 1

Установил виртуалку pg-01 с Ubuntu 24.04.2 LTS.
Поставил на нее Postgresql 17:

    postgres=# select version();
                                                                  version
    -----------------------------------------------------------------------------------------------------------------------------------
     PostgreSQL 17.5 (Ubuntu 17.5-1.pgdg24.04+1) on x86_64-pc-linux-gnu, compiled by gcc (Ubuntu 13.3.0-6ubuntu2~24.04) 13.3.0, 64-bit
    (1 row)

+

    root@pg-01:~# pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    17  main    5432 online postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

Создал табличку:

    postgres=# create table test(c1 text);
    CREATE TABLE
    postgres=# insert into test values('1');
    INSERT 0 1

Кластер остановлен:

    pg_ctlcluster 17 main stop
    root@pg-01:~# pg_lsclusters
    Ver Cluster Port Status Owner    Data directory              Log file
    17  main    5432 down   postgres /var/lib/postgresql/17/main /var/log/postgresql/postgresql-17-main.log

## Часть 2

Примонтировал новый диск /dev/sdb1:

    root@pg-01:~# df -h
    Filesystem                         Size  Used Avail Use% Mounted on
    tmpfs                              392M  1.1M  391M   1% /run
    efivarfs                           256K   55K  197K  22% /sys/firmware/efi/efivars
    /dev/mapper/ubuntu--vg-ubuntu--lv   11G  4.8G  5.4G  48% /
    tmpfs                              2.0G     0  2.0G   0% /dev/shm
    tmpfs                              5.0M     0  5.0M   0% /run/lock
    /dev/sda2                          2.0G  101M  1.7G   6% /boot
    /dev/sda1                          1.1G  6.2M  1.1G   1% /boot/efi
    tmpfs                              392M   12K  392M   1% /run/user/1000
    /dev/sdb1                          9.8G   24K  9.3G   1% /opt/10g

Остановил кластер и переместил директорию с данными:

    root@pg-01:~# mv /var/lib/postgresql/17/main /opt/10g

Проверка:

    root@pg-01:~# ls /opt/10g/main -al
    total 88
    drwx------ 19 postgres postgres 4096 Jun 17 19:59 .
    drwxr-xr-x  3 postgres postgres 4096 Jun 17 19:59 ..
    drwx------  5 postgres postgres 4096 Jun 17 19:31 base
    drwx------  2 postgres postgres 4096 Jun 17 19:56 global
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_commit_ts
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_dynshmem
    drwx------  4 postgres postgres 4096 Jun 17 19:59 pg_logical
    drwx------  4 postgres postgres 4096 Jun 17 19:31 pg_multixact
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_notify
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_replslot
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_serial
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_snapshots
    drwx------  2 postgres postgres 4096 Jun 17 19:59 pg_stat
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_stat_tmp
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_subtrans
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_tblspc
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_twophase
    -rw-------  1 postgres postgres    3 Jun 17 19:31 PG_VERSION
    drwx------  4 postgres postgres 4096 Jun 17 19:31 pg_wal
    drwx------  2 postgres postgres 4096 Jun 17 19:31 pg_xact
    -rw-------  1 postgres postgres   88 Jun 17 19:31 postgresql.auto.conf
    -rw-------  1 postgres postgres  130 Jun 17 19:55 postmaster.opts

Кластер разумеется не стартует:

    root@pg-01:~# pg_ctlcluster 17 main start
    Error: /var/lib/postgresql/17/main is not accessible or does not exist

Однако если внести изменения в файл /etc/postgresql/17/main/postgresql.conf:

    data_directory = '/opt/10g/main'
    #data_directory = '/var/lib/postgresql/17/main'

Кластер стартует:

    root@pg-01:~# pg_ctlcluster 17 main start
    root@pg-01:~# pg_lsclusters
    Ver Cluster Port Status Owner    Data directory Log file
    17  main    5432 online postgres /opt/10g/main  /var/log/postgresql/postgresql-17-main.log

Данные в таблице на месте:

    postgres=# select * from test;
    c1
    ----
    1
    (1 row)

## Часть 3

Перемонтировали существующий диск ко второму серверу:

    root@pg-02:~# mkdir /opt/lalala
    root@pg-02:~# mount --source /dev/sdb1 --target /opt/lalala/
    root@pg-02:~# ls /opt/lalala/main/
    base    pg_commit_ts  pg_logical    pg_notify    pg_serial     pg_stat      pg_subtrans  pg_twophase  pg_wal   postgresql.auto.conf  postmaster.pid
    global  pg_dynshmem   pg_multixact  pg_replslot  pg_snapshots  pg_stat_tmp  pg_tblspc    PG_VERSION   pg_xact  postmaster.opts

Кластер успешно стартовал:

    root@pg-02:~# pg_ctlcluster 17 main start
    root@pg-02:~# pg_lsclusters
    Ver Cluster Port Status Owner    Data directory   Log file
    17  main    5432 online postgres /opt/lalala/main /var/log/postgresql/postgresql-17-main.log

Данные в таблице целые:

    postgres=# select * from test;
    c1
    ----
    1
    (1 row)

По большому счету часть 3 ничем не отличается от части 2...