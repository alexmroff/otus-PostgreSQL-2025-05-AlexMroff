# Домашнее задание к уроку №18 - Триггеры.

На основе предложенного скрипта **hw_triggers.sql** создана БД **triggers**.

## Начальное заполнение витрины

Производим начальное заполнение витрины на основе данных в таблицах:

    triggers=# INSERT INTO good_sum_mart
    SELECT DISTINCT g.good_name, sum (s.sales_qty * g.good_price) OVER (PARTITION BY s.good_id)
    FROM goods g INNER JOIN sales s ON s.good_id = g.goods_id;
    INSERT 0 2
    triggers=# SELECT * FROM good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Спички хозайственные     |        65.50
    Автомобиль Ferrari FXX K | 185000000.01
    (2 rows)

## Триггерная функция

Одна триггерная функция для всех триггеров, учитывающая в том числе появление товаров, которых до этого не было в витрине.

    CREATE OR REPLACE FUNCTION pract_functions.sales_trigger() RETURNS trigger AS 
    $$
        BEGIN
            IF TG_OP = 'INSERT' THEN
                -- Если товар уже есть в витрине
                IF EXISTS 
                    (SELECT *
                    FROM pract_functions.goods g INNER JOIN pract_functions.good_sum_mart gsm
                    ON gsm.good_name = g.good_name
                    WHERE NEW.good_id = g.goods_id
                    )
                THEN
                    WITH good_values (good_name, good_price) AS
                        (SELECT g.good_name, g.good_price FROM pract_functions.goods g
                        WHERE g.goods_id = NEW.good_id)
                    UPDATE pract_functions.good_sum_mart gsm
                        SET sum_sale = sum_sale + good_values.good_price * NEW.sales_qty
                        FROM good_values
                        WHERE gsm.good_name = good_values.good_name;
                    RETURN NEW;
                -- Если это новый товар, которого в витрине пока нет
                ELSE
                    WITH good_values (good_name, good_price) AS
                        (SELECT g.good_name, g.good_price FROM pract_functions.goods g
                        WHERE g.goods_id = NEW.good_id)
                    INSERT INTO pract_functions.good_sum_mart (good_name, sum_sale)
                    SELECT good_name, good_price * NEW.sales_qty
                    FROM good_values;
                    RETURN NEW;
                END IF;
            -- Если запись о продаже обновилась
            ELSIF TG_OP = 'UPDATE' THEN
                WITH good_values (good_name, good_price) AS
                    (SELECT g.good_name, g.good_price FROM pract_functions.goods g
                    WHERE g.goods_id = NEW.good_id)
                UPDATE pract_functions.good_sum_mart gsm
                    SET sum_sale = sum_sale + good_values.good_price * (NEW.sales_qty - OLD.sales_qty)
                    FROM good_values
                    WHERE gsm.good_name = good_values.good_name;			
                RETURN NEW;
            -- Если запись о продаже удалилась
            ELSIF TG_OP = 'DELETE' THEN
                WITH good_values (good_name, good_price) AS
                    (SELECT g.good_name, g.good_price FROM pract_functions.goods g
                    WHERE g.goods_id = OLD.good_id)
                UPDATE pract_functions.good_sum_mart gsm
                    SET sum_sale = sum_sale - good_values.good_price * OLD.sales_qty
                    FROM good_values
                    WHERE gsm.good_name = good_values.good_name;			
                RETURN OLD;
            END IF;	
        END;
    $$
    LANGUAGE 'plpgsql';

## Триггеры

    CREATE OR REPLACE TRIGGER sales_insert
    BEFORE INSERT ON pract_functions.sales
    FOR EACH ROW
    EXECUTE FUNCTION pract_functions.sales_trigger();

    CREATE OR REPLACE TRIGGER sales_update
    BEFORE UPDATE ON pract_functions.sales
    FOR EACH ROW
    EXECUTE FUNCTION pract_functions.sales_trigger();

    CREATE OR REPLACE TRIGGER sales_delete
    AFTER DELETE ON pract_functions.sales
    FOR EACH ROW
    EXECUTE FUNCTION pract_functions.sales_trigger();

## Тесты и результаты

Фиксируем начальное состояние системы:

    triggers=# SELECT * FROM pract_functions.goods;
    goods_id |        good_name         |  good_price
    ----------+--------------------------+--------------
            1 | Спички хозайственные     |         0.50
            2 | Автомобиль Ferrari FXX K | 185000000.01
            3 | Масло                    |         5.00
    (3 rows)

    triggers=# DELETE FROM pract_functions.goods WHERE goods_id = 3;
    DELETE 1
    triggers=#
    triggers=#
    triggers=#
    triggers=# SELECT * FROM pract_functions.goods;
    goods_id |        good_name         |  good_price
    ----------+--------------------------+--------------
            1 | Спички хозайственные     |         0.50
            2 | Автомобиль Ferrari FXX K | 185000000.01
    (2 rows)

    triggers=# SELECT * FROM pract_functions.sales;
    sales_id | good_id |          sales_time           | sales_qty
    ----------+---------+-------------------------------+-----------
        21 |       1 | 2025-08-11 17:54:43.369477+00 |        10
        22 |       1 | 2025-08-11 17:54:43.369477+00 |         1
        23 |       1 | 2025-08-11 17:54:43.369477+00 |       120
        24 |       2 | 2025-08-11 17:54:43.369477+00 |         1
    (4 rows)

    triggers=# SELECT * FROM pract_functions.good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Спички хозайственные     |        65.50
    Автомобиль Ferrari FXX K | 185000000.01
    (2 rows)

Добавляем в sales продажу существующего товара, фиксируем срабатывание триггера:

    triggers=# INSERT INTO pract_functions.sales (good_id, sales_qty) VALUES (1, 2);
    INSERT 0 1
    triggers=# SELECT * FROM pract_functions.good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Автомобиль Ferrari FXX K | 185000000.01
    Спички хозайственные     |        66.50
    (2 rows)

Добавляем новый товар в goods, продаем его в sales, фиксируем срабатывание триггера:

    triggers=# INSERT INTO pract_functions.goods (goods_id, good_name, good_price)
    VALUES (5, 'Масло', 5);
    INSERT 0 1
    triggers=# INSERT INTO pract_functions.sales (good_id, sales_qty) VALUES (5, 2);
    INSERT 0 1
    triggers=# SELECT * FROM pract_functions.good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Автомобиль Ferrari FXX K | 185000000.01
    Спички хозайственные     |        66.50
    Масло                    |        10.00
    (3 rows)

Изменяем данные количества в последней продаже:

    triggers=# SELECT * FROM pract_functions.sales order by sales_id;
    sales_id | good_id |          sales_time           | sales_qty
    ----------+---------+-------------------------------+-----------
        21 |       1 | 2025-08-11 17:54:43.369477+00 |        10
        22 |       1 | 2025-08-11 17:54:43.369477+00 |         1
        23 |       1 | 2025-08-11 17:54:43.369477+00 |       120
        24 |       2 | 2025-08-11 17:54:43.369477+00 |         1
        26 |       1 | 2025-08-11 17:59:29.778824+00 |         2
        27 |       5 | 2025-08-11 18:01:01.854431+00 |         2
    (6 rows)

    triggers=# UPDATE pract_functions.sales SET sales_qty = 4 WHERE sales_id = 27;
    UPDATE 1
    triggers=# SELECT * FROM pract_functions.good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Автомобиль Ferrari FXX K | 185000000.01
    Спички хозайственные     |        66.50
    Масло                    |        20.00
    (3 rows)

Удаляем продажу с sales_id = 21, фиксируем срабатывание триггера:

    triggers=# DELETE FROM pract_functions.sales WHERE sales_id = 21;
    DELETE 1
    triggers=# SELECT * FROM pract_functions.good_sum_mart;
            good_name         |   sum_sale
    --------------------------+--------------
    Автомобиль Ferrari FXX K | 185000000.01
    Масло                    |        20.00
    Спички хозайственные     |        61.50
    (3 rows)

Задача решена!

## Задание со звездочкой

Смотрим на формат отчета из приведенного скрипта:

    SELECT G.good_name, sum(G.good_price * S.sales_qty)
    FROM goods G
    INNER JOIN sales S ON S.good_id = G.goods_id
    GROUP BY G.good_name;

Легко видеть, что при генерации отчета все количества продаж будут умножаться на **текущую** цену товара.
В случае если ранее товар продавался по другой цене, в отчете это учтено не будет.

Наша же витрина данных это учитывает!