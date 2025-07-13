# Домашнее задание к уроку №11 - Выборка данных, виды join'ов. Применение и оптимизация.

Для выполнения задания использую демо-базу https://edu.postgrespro.ru/demo_small.zip
Схема таблиц - в соседнем файле.

## Реализовать прямое соединение двух или более таблиц

Посмотрим все места во всех самолётах:

    demo=# select ad.aircraft_code, ad.model, ad.range, s.seat_no, s.fare_conditions
    from aircrafts_data ad
    inner join seats s on ad.aircraft_code = s.aircraft_code
    limit 10;

    aircraft_code |                        model                        | range | seat_no | fare_conditions
    ---------------+-----------------------------------------------------+-------+---------+-----------------
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 2A      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 2C      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 2D      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 2F      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 3A      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 3C      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 3D      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 3F      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 4A      | Business
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 4C      | Business
    (10 rows)

## Реализовать левостороннее (или правостороннее) соединение двух или более таблиц

Посмотрим все аэропорты, а также относящие к ним рейсы, если таковые есть:

    demo=# select a.airport_code, a.city, a.timezone, f.flight_id, f.flight_no, f.departure_airport, f.arrival_airport, f.status
    from airports_data a
    left join flights f
    on a.airport_code = f.arrival_airport
    order by a.airport_code
    limit 10;

    airport_code |              city              |   timezone    | flight_id | flight_no | departure_airport | arrival_airport |  status
    --------------+--------------------------------+---------------+-----------+-----------+-------------------+-----------------+-----------
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6357 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6358 | PG0251    | SVO               | AAQ             | Scheduled
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6359 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6360 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6361 | PG0251    | SVO               | AAQ             | Scheduled
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6362 | PG0251    | SVO               | AAQ             | Scheduled
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6363 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6364 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6365 | PG0251    | SVO               | AAQ             | Arrived
    AAQ          | {"en": "Anapa", "ru": "Анапа"} | Europe/Moscow |      6366 | PG0251    | SVO               | AAQ             | Scheduled
    (10 rows)

## Реализовать кросс соединение двух или более таблиц

Пример без великого практического смысла, но зато реализующий CROSS JOIN - декартово произведение всех кодов самолётов и аэропортов отправления:

    demo=# select distinct a.aircraft_code, f.departure_airport
    from aircrafts_data a
    join flights f on a.aircraft_code = f.aircraft_code
    order by a.aircraft_code, f.departure_airport
    limit 10;
    aircraft_code | departure_airport
    ---------------+-------------------
    319           | ABA
    319           | AER
    319           | ARH
    319           | ASF
    319           | BTK
    319           | CNN
    319           | DME
    319           | DYR
    319           | HTA
    319           | IKT
    (10 rows)

## Реализовать полное соединение двух или более таблиц

    demo=# select ad.aircraft_code, ad.model, ad.range, s.seat_no, s.fare_conditions
    from aircrafts_data ad
    full outer join seats s on ad.aircraft_code = s.aircraft_code
    limit 10;
    aircraft_code |                        model                        | range | seat_no | fare_conditions
    ---------------+-----------------------------------------------------+-------+---------+-----------------
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10A     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10B     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10C     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10D     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10E     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 10F     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 11A     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 11B     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 11C     | Economy
    319           | {"en": "Airbus A319-100", "ru": "Аэробус A319-100"} |  6700 | 11D     | Economy
    (10 rows)

## Реализовать запрос, в котором будут использованы разные типы соединений

В таком запросе будут NULL в двух последних столбцах, если бронирование было совершено, билет куплен, но полет не состоялся.

    demo=# select b.book_date, t.ticket_no, t.passenger_id, t.passenger_name, tf.flight_id, f.departure_airport, f.arrival_airport
    from bookings b
    inner join tickets t on b.book_ref = t.book_ref
    inner join ticket_flights tf on t.ticket_no = tf.ticket_no
    left join flights f on tf.flight_id = f.flight_id
    limit 10;
        book_date        |   ticket_no   | passenger_id |   passenger_name    | flight_id | departure_airport | arrival_airport
    ------------------------+---------------+--------------+---------------------+-----------+-------------------+-----------------
    2017-07-05 17:19:00+00 | 0005432000987 | 8149 604011  | VALERIY TIKHONOV    |     28935 | CSY               | SVO
    2017-07-05 17:19:00+00 | 0005432000988 | 8499 420203  | EVGENIYA ALEKSEEVA  |     28935 | CSY               | SVO
    2017-06-28 22:55:00+00 | 0005432000989 | 1011 752484  | ARTUR GERASIMOV     |     28939 | CSY               | SVO
    2017-06-28 22:55:00+00 | 0005432000990 | 4849 400049  | ALINA VOLKOVA       |     28939 | CSY               | SVO
    2017-07-03 01:37:00+00 | 0005432000991 | 6615 976589  | MAKSIM ZHUKOV       |     28913 | CSY               | SVO
    2017-07-03 01:37:00+00 | 0005432000992 | 2021 652719  | NIKOLAY EGOROV      |     28913 | CSY               | SVO
    2017-07-03 01:37:00+00 | 0005432000993 | 0817 363231  | TATYANA KUZNECOVA   |     28913 | CSY               | SVO
    2017-07-07 00:03:00+00 | 0005432000994 | 2883 989356  | IRINA ANTONOVA      |     28912 | CSY               | SVO
    2017-07-07 00:03:00+00 | 0005432000995 | 3097 995546  | VALENTINA KUZNECOVA |     28912 | CSY               | SVO
    2017-07-05 21:08:00+00 | 0005432000996 | 6866 920231  | POLINA ZHURAVLEVA   |     28929 | CSY               | SVO
    (10 rows)
