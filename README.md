# Домашнее задание к занятию «SQL. Часть 1»

---

Задание можно выполнить как в любом IDE, так и в командной строке.

### Задание 1

Получите уникальные названия районов из таблицы с адресами, которые начинаются на “K” и заканчиваются на “a” и не содержат пробелов.  

### Решение:    

- Задания будут выполнены в командной строке непосредственно в контейнере с MySQL. Зайдем в контейнер, используя команду

```
docker exec -it fa25621f8469 mysql -u root -p
```

- Загрузим дамп базы sakila и добавим ее в нашу созданную пустую базу sakila:

```
bash-4.4# mysql -u root -p sakila < sakila-schema.sql
Enter password:
bash-4.4# mysql -u root -p sakila < sakila-data.sql
Enter password:
```

- Проверим, что база подгрузилась и выполним необходимый запрос:

```
SELECT DISTINCT district FROM address WHERE district LIKE 'K%a' AND district NOT LIKE '% %';
+-----------+
| district  |
+-----------+
| Kanagawa  |
| Kalmykia  |
| Kaduna    |
| Karnataka |
| K▒tahya   |
| Kerala    |
| Kitaa     |
+-----------+
7 rows in set (0.00 sec)

```

---

### Задание 2  

Получите из таблицы платежей за прокат фильмов информацию по платежам, которые выполнялись в промежуток с 15 июня 2005 года по 18 июня 2005 года **включительно** и стоимость которых превышает 10.00.  

### Решение:    

- Выполним запрос, который будет соответствовать заданию:

```
mysql> SELECT amount, payment_date FROM payment WHERE payment_date BETWEEN '2005-06-15 00:00:00' AND '2005-06-18 23:59:59' and amount > 10 ORDER BY payment_date;
+--------+---------------------+
| amount | payment_date        |
+--------+---------------------+
|  10.99 | 2005-06-15 09:46:33 |
|  10.99 | 2005-06-15 18:30:46 |
|  10.99 | 2005-06-16 14:52:02 |
|  10.99 | 2005-06-17 04:05:12 |
|  10.99 | 2005-06-17 18:09:04 |
|  11.99 | 2005-06-17 23:51:21 |
|  10.99 | 2005-06-18 08:33:23 |
+--------+---------------------+
7 rows in set (0.03 sec)
```

---

### Задание 3    

Получите последние пять аренд фильмов.  

### Решение:    

- Здесь, для своего же понимания, выполняем все наглядно, без алиасов и т.д.:

```
mysql> SELECT sakila.film.title, sakila.rental.rental_date
    -> FROM sakila.rental, sakila.inventory, sakila.film
    -> WHERE sakila.rental.inventory_id = sakila.inventory.inventory_id
    -> AND sakila.inventory.film_id = sakila.film.film_id
    -> ORDER BY sakila.rental.rental_date DESC
    -> LIMIT 5;
+--------------------+---------------------+
| title              | rental_date         |
+--------------------+---------------------+
| ZHIVAGO CORE       | 2006-02-14 15:16:03 |
| WORLD LEATHERNECKS | 2006-02-14 15:16:03 |
| WOMEN DORADO       | 2006-02-14 15:16:03 |
| WINDOW SIDE        | 2006-02-14 15:16:03 |
| WILD APOLLO        | 2006-02-14 15:16:03 |
+--------------------+---------------------+
5 rows in set (0.00 sec)
```

---

### Задание 4

Одним запросом получите активных покупателей, имена которых Kelly или Willie. 

Сформируйте вывод в результат таким образом:
- все буквы в фамилии и имени из верхнего регистра переведите в нижний регистр,
- замените буквы 'll' в именах на 'pp'.

### Решение:    

- Это задание вызвало у меня некоторые трудности и пришлось гуглить. Получился следующий запрос:

```
mysql> SELECT
    ->   LOWER(first_name) AS first_name,
    ->   REPLACE(LOWER(last_name), 'll', 'pp') AS last_name
    -> FROM
    ->   sakila.customer
    -> WHERE
    ->   first_name = 'kelly' OR first_name = 'willie';
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| kelly      | torres    |
| willie     | howepp    |
| willie     | markham   |
| kelly      | knott     |
+------------+-----------+
4 rows in set (0.00 sec)
```

- Здесь мы используем алиасы, чтобы вывести "исправленные" таблицы first_name и last_name (в нижнем регистре), а также две функции длы запроса к last_name, т.к. нам нужно вывести имя в нижгнем регистре (LOWER) и заменить двойные l на двойные p (REPLACE).

Проверим, убрав REPLACE, случилась ли вообще замена:  

```
mysql> SELECT
    ->   LOWER(first_name) AS first_name,
    ->   LOWER(last_name) AS last_name
    -> FROM
    ->   sakila.customer
    -> WHERE
    ->   first_name IN ('kelly', 'willie');
+------------+-----------+
| first_name | last_name |
+------------+-----------+
| kelly      | torres    |
| willie     | howell    |
| willie     | markham   |
| kelly      | knott     |
+------------+-----------+
4 rows in set (0.01 sec)
```

Да, все супер.

---
