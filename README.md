# Домашнее задание к занятию «SQL. Часть 1»

---

Задание можно выполнить как в любом IDE, так и в командной строке.  


### Предисловие  

- База данных была развернута при помощи Ansible. Я использовал ту же машину, что использую для других Ansible-ролей. MySQL будет развернута в контейнере при помощи docker-compose.

- Tasks:

```
---
#удаляем старые версии доккера, если есть
     - name: remove docker old versions
       apt:
         pkg:
           - docker
           - docker-engine
           - docker.io
           - containerd
           - runc
         state: absent

#Устанавливаем необходимые предварительные пакеты
     - name: Install pre-requisites
       apt:
         pkg:
           - ca-certificates
           - curl
           - gnupg
           - lsb-release
           - apt-transport-https
         state: latest
         update_cache: yes

#Идем далее по инструкции, добавляем GPG ключ для доккера
     - name: Add Docker GPG apt Key
       apt_key:
         url: "https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg"
         state: present

#Добавляем репозиторий
     - name: Add Docker Repository
       apt_repository:
         repo: "{{ docker_apt_repository }}"
         state: present
         filename: docker
         update_cache: true

#Я решил оставить это и следующее действие, т.к. изначально у меня возникли трудности с установкой доккер + доккер компоуз на Debian 11 (опишу в пояснительной).
     - name: Try command
       command: echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.$

#Это я еще раз апдейтил кэш после выполнения предыдущей команды. По идее, эта и предыдущая команда в этом плэйбуке лишние.
     - name: apt update
       apt:
         update_cache: true

#Устанавливаем пакеты доккера
     - name: Update apt and install docker-ce
       apt:
         pkg:
           - docker-ce
           - docker-ce-cli
           - containerd.io
           - docker-buildx-plugin
           - docker-compose-plugin
         state: latest
         update_cache: false

#Рестартуем службу, ну и инэйблд = йес, само-собой
     - name: Starting Docker service
       service:
         name: docker
         state: restarted
         enabled: yes

#Ставим доккер-компоуз
     - name: Install docker-compose
       get_url:
         url : "https://github.com/docker/compose/releases/download/{{ docker_compose_version }}/docker-compose-Linux-x86_64"
         dest: /usr/local/bin/docker-compose
         mode: 'a+x'
         force: yes

#Создаем тимплэйт docker-compose.yaml файла
     - name: Create and fill docker-compose file
       template:
         src: /home/vagrant/vita_proj/roles/vita_docker/templates/docker_compose_sql.j2
         dest: /home/vagrant/sql/docker-compose.yaml

#Запускаем все это дело на удаленной машине
     - name: Starting docker-compose on remote machine
       command: docker compose up -d
       args:
         chdir: /home/vagrant/sql
```

- Templates (docker-compose.yaml):

```
version: '3.1'

services:
  mysql_db:
    image: mysql:8
    restart: always
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: sakila
      MYSQL_USER: admin
      MYSQL_PASSWORD: strongpassword
    volumes:
      - ./dbdata:/var/lib/mysql/
      - ./sakila-db:/tmp/mysql/backup
```

- Defaults:

```
---
docker_apt_release_channel: stable
docker_apt_arch: amd64
docker_apt_repository: "deb [arch={{ docker_apt_arch }}] https://download.docker.com/linux/{{ ansible_distribution | lower }} {{ ansible_distribution_release }} {{ docker_apt_release_channel }}"
docker_apt_gpg_key: https://download.docker.com/linux/{{ ansible_distribution | lower }}/gpg
docker_compose_version: "1.24.0"
```


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

- Здесь, для своего же понимания, выполняем все наглядно, без алиасов, джойнов и т.д.:

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

- На самом деле, это не самое простое задание. Теперь нам нужно получить данные из трех баз, соответственно, нам нужно их связать (rental, inventory, film) по общим столбцам, тогдда, при сравнении данных, запрос выведет нам правильные значения. Также, мы сортируем (ORDER BY) по rental_date в обратном порядке (DESC), что требуется нам по условию.

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

### Задание 5*

Выведите Email каждого покупателя, разделив значение Email на две отдельных колонки: в первой колонке должно быть значение, указанное до @, во второй — значение, указанное после @.  

### Решение:  

- Решил выполнить это задание, потому что оно показалось мне не очень сложным. Нужно лишь найти парвильную функцию и вот она (очень интересная, кстати) - **SUBSTRING_INDEX**.  
Функция позволяет извлечь нужную нам подстроку из строки:  

```
mysql> SELECT
    ->   SUBSTRING_INDEX(email, '@', 1) AS email_username,
    ->   SUBSTRING_INDEX(email, '@', -1) AS email_domain
    -> FROM
    ->   sakila.customer
    -> ORDER BY RAND()
    -> LIMIT 10;
+-----------------+--------------------+
| email_username  | email_domain       |
+-----------------+--------------------+
| CURTIS.IRBY     | sakilacustomer.org |
| BRANDY.GRAVES   | sakilacustomer.org |
| LUCILLE.HOLMES  | sakilacustomer.org |
| EDITH.MCDONALD  | sakilacustomer.org |
| CINDY.FISHER    | sakilacustomer.org |
| ROSE.HOWARD     | sakilacustomer.org |
| RANDALL.NEUMANN | sakilacustomer.org |
| CARMEN.OWENS    | sakilacustomer.org |
| WILMA.RICHARDS  | sakilacustomer.org |
| ALVIN.DELOACH   | sakilacustomer.org |
+-----------------+--------------------+
10 rows in set (0.00 sec)
```

- Что мы тут видим. **SUBSTRING_INDEX(email, '@', 1)**, email - разделяемый столбец, @ - разделитель, 1 и -1 показывает, какую часть мы забираем - до разделителя или после. Плюс, я еще немного доработал запрос и он у меня выводит 10 рандомных записей (чтобы всех не выводить).


---

