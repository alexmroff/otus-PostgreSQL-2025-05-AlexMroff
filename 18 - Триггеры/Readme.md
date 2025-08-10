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
                -- Если такой товар в витрине уже есть - обновляем сумму
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
                -- Если такого товара в витрине еще нет - добавляем его вместе с суммой
                ELSE
                    WITH good_values (good_name, good_price) AS
                        (SELECT g.good_name, g.good_price FROM pract_functions.goods g
                        WHERE g.goods_id = NEW.good_id)
                    INSERT INTO pract_functions.good_sum_mart (good_name, sum_sale)
                    SELECT good_name, good_price * NEW.sales_qty
                    FROM good_values;
                    RETURN NEW;
                END IF;
            ELSIF TG_OP = 'UPDATE' THEN
                RETURN NULL;
            ELSIF TG_OP = 'DELETE' THEN
                RETURN NULL;
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
    BEFORE DELETE ON pract_functions.sales
    FOR EACH ROW
    EXECUTE FUNCTION pract_functions.sales_trigger();

## Дополнительный код на завтра

    SELECT * FROM pract_functions.good_sum_mart
    SELECT * FROM pract_functions.sales
    SELECT * FROM pract_functions.goods
    DELETE FROM good_sum_mart

    INSERT INTO good_sum_mart
    SELECT DISTINCT g.good_name, sum (s.sales_qty * g.good_price) OVER (PARTITION BY s.good_id)
    FROM pract_functions.goods g INNER JOIN pract_functions.sales s ON s.good_id = g.goods_id;

    INSERT INTO pract_functions.sales
    (good_id, sales_qty) VALUES (3, 5)

    INSERT INTO pract_functions.goods (goods_id, good_name, good_price)
    VALUES (3, 'Масло', 5)