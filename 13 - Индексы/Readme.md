# Домашнее задание к уроку №13 - Виды индексов. Работа с индексами и оптимизация запросов.

Работаем с той же БД demo, что и в прошлом ДЗ.
На примере таблицы **flights**:

    demo=# select count (*) from flights;
    count
    -------
    33121
    (1 row)

    demo=# \d flights
                                                Table "bookings.flights"
        Column        |           Type           | Collation | Nullable |                  Default
    ---------------------+--------------------------+-----------+----------+--------------------------------------------
    flight_id           | integer                  |           | not null | nextval('flights_flight_id_seq'::regclass)
    flight_no           | character(6)             |           | not null |
    scheduled_departure | timestamp with time zone |           | not null |
    scheduled_arrival   | timestamp with time zone |           | not null |
    departure_airport   | character(3)             |           | not null |
    arrival_airport     | character(3)             |           | not null |
    status              | character varying(20)    |           | not null |
    aircraft_code       | character(3)             |           | not null |
    actual_departure    | timestamp with time zone |           |          |
    actual_arrival      | timestamp with time zone |           |          |
    Indexes:
        "flights_pkey" PRIMARY KEY, btree (flight_id)
        "flights_flight_no_scheduled_departure_key" UNIQUE CONSTRAINT, btree (flight_no, scheduled_departure)
    Check constraints:
        "flights_check" CHECK (scheduled_arrival > scheduled_departure)
        "flights_check1" CHECK (actual_arrival IS NULL OR actual_departure IS NOT NULL AND actual_arrival IS NOT NULL AND actual_arrival > actual_departure)
        "flights_status_check" CHECK (status::text = ANY (ARRAY['On Time'::character varying::text, 'Delayed'::character varying::text, 'Departed'::character varying::text, 'Arrived'::character varying::text, 'Scheduled'::character varying::text, 'Cancelled'::character varying::text]))
    Foreign-key constraints:
        "flights_aircraft_code_fkey" FOREIGN KEY (aircraft_code) REFERENCES aircrafts_data(aircraft_code)
        "flights_arrival_airport_fkey" FOREIGN KEY (arrival_airport) REFERENCES airports_data(airport_code)
        "flights_departure_airport_fkey" FOREIGN KEY (departure_airport) REFERENCES airports_data(airport_code)
    Referenced by:
        TABLE "ticket_flights" CONSTRAINT "ticket_flights_flight_id_fkey" FOREIGN KEY (flight_id) REFERENCES flights(flight_id)

## Создать индекс к какой-либо из таблиц вашей БД
Это пример самого простого btree-индекса.

    demo=# create index idx_flight_no on flights (flight_no);
    CREATE INDEX

## Прислать текстом результат команды explain, в которой используется данный индекс

    demo=# explain select * from flights where flight_no = 'PG0003';
                                    QUERY PLAN
    -----------------------------------------------------------------------------
    Bitmap Heap Scan on flights  (cost=4.41..55.79 rows=15 width=63)
    Recheck Cond: (flight_no = 'PG0003'::bpchar)
    ->  Bitmap Index Scan on idx_flight_no  (cost=0.00..4.40 rows=15 width=0)
            Index Cond: (flight_no = 'PG0003'::bpchar)
    (4 rows)

## Реализовать индекс для полнотекстового поиска

Синтетический пример GIN-индекса с jsonb:

    demo=# CREATE TABLE test (
    id bigserial PRIMARY KEY,
    data jsonb
    );
    INSERT INTO test(data) VALUES ('{"field": "value1"}');
    INSERT INTO test(data) VALUES ('{"field": "value2"}');
    INSERT INTO test(data) VALUES ('{"other_field": "value42"}');
    CREATE INDEX ON test USING gin(data);
    CREATE TABLE
    INSERT 0 1
    INSERT 0 1
    INSERT 0 1
    CREATE INDEX

## Реализовать индекс на часть таблицы или индекс на поле с функцией

Синтетический пример:

    demo=# create index idx_func_flights on flights (lower(status));
    CREATE INDEX

## Создать индекс на несколько полей

И снова синтетический пример:

    demo=# create index idx_multi_field on flights (departure_airport, arrival_airport);
    CREATE INDEX

Всё :)