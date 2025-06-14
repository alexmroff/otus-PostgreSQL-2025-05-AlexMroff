# Домашнее задание к уроку №3 - Установка PostgreSQL

Для установки PostgreSQL 15 через Docker используется та же виртуалка с Ubuntu 24.04, на которой уже установлен PostgreSQL 16.9.
В связи с этим для docker-версии PostgreSQL 15 используется нестандартный порт 5433.

## Установка Docker Engine

Установка выполнена согласно инструкции: https://docs.docker.com/engine/install/ubuntu/

## Особенности установки PostgreSQL

1. Для установки используется формат yaml для запуска через Docker Compose;
2. На хостовой машине создана директория */pgdata*, которая пробрасывается в контейнер внутри *compose.yaml*
3. Для того, чтобы PostgreSQL внутри контейнера был доступен по сети после установки в директорию */pgdata* помещён файл *pg_hba.conf*, в котором пользователю **postgres** разрещшено подключение по сети: `host 	all		postgres	192.168.155.0/24	trust`
4. Пароль пользователя **postgres** в данном примере хранится в *compose.yaml* в открытом виде. Разумеется, в продуктивной среде такое решение использовать нельзя.
5. Контейнер запускается командой: `sudo docker compose up -d`

## Результаты тестирования

После настройки PostgreSQL внутри Docker вышеуказанным способом в БД postgres была создана тестовая табличка:

`
CREATE TABLE products (
    product_no integer,
    name text,
    price numeric
);

INSERT INTO products (product_no, name, price) VALUES (1, 'Cheese', 9.99);
INSERT INTO products (name, price, product_no) VALUES ('Milk', 5.99, 2);
`

После этого контейнер был удалён: `sudo docker compose down`

Команда `sudo docker ps -a` показывает, что на данный момент ни запущенных, ни остановленных контейнеров в системе нет.

После этого контейнер создан заново той же командой: `sudo docker compose up -d`

С помощью команды `SELECT * FROM products;` можно считать из таблицы products данные, которые были записаны в нее во время существования предыдущего контейнера.

Задача решена!

🤩🤩🤩🤩🤩

