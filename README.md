# otus_work
Выполнение заданий OTUS

--Создание пары ssh ключа
>ssh-keygen -t ed25519

--скопировал ключ
>type C:\Users\mekom\.ssh\id_ed25519.pub | clip

--Мой ключ 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPrylJAe3RK0Nqir6Hav1X+OxiztES3oKRYy9jdS2JQY mekom@LAPTOP-UIR319FF

--Подключение к ВМ
>ssh -i ~/.ssh/id_ed25519 komarov@158.160.130.54

--Установка PG
C:\Users\mekom>ssh -i ~/.ssh/id_ed25519 komarov@158.160.128.48
Welcome to Ubuntu 20.04.6 LTS (GNU/Linux 5.4.0-170-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/pro
New release '22.04.3 LTS' available.
Run 'do-release-upgrade' to upgrade to it.

Last login: Sun Feb 18 04:24:54 2024 from 178.46.77.138

komarov@komarov-vm:~$ sudo apt update && sudo apt upgrade -y -q && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt -y install postgresql-15


--Проверяем установленный кластер 
komarov@komarov-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log

--Подключаемся к PG
komarov@komarov-vm:~$ sudo -u postgres psql
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

--Переходим к настройке конфигурации
\q

--В терминале ВМ
> sudo nano /etc/postgresql/15/main/postgresql.conf

Вопрос:
IP адрес мы должны указать с которого будем подключаться к серверу, с ноутбука например?

>sudo nano /etc/postgresql/15/main/pg_hba.conf

Вопрос:
Зачем ipv4  прописываем 0.0.0.0./0


--Рестар кластера
sudo pg_ctlcluster 15 main restart

--Создаем пользователя для внешнего подключения
sudo -u postgres psql

postgres=# create role komarov password 'komarov' login; \du
postgres=# create database otus; \l
postgres=# \q

--Делаем рестарт кластера
komarov@komarov-vm:~$ sudo pg_ctlcluster 15 main restart

--Подключаемся из вне 
Я выбрал подключиться с ноутбука в Dbeaver  

В connection settings указал подключение по хосту 
Host: 158.160.130.54
Port: 5432
Database: otus
username: komarov
password: komarov

Подключено успешно!

--Удаляем кластера
komarov@komarov-vm:~$ sudo pg_ctlcluster 15 main stop && sudo pg_dropcluster 15 main
komarov@komarov-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner Data directory Log file

--Устанавливаем Докер:
komarov@komarov-vm:~$ curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker

--СОздаем Докер сеть
komarov@komarov-vm:~$ sudo docker network create pg-net
Идентификатор сети
d0c478864630b85c87fb65c48e74ecb447dc2f6e042fbbf53d02513f9efa8de9

-- Подключаем созданный контейннер к серверу Pg
komarov@komarov-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15 

--Запускаем отдельный контейнер с клиентом
komarov@komarov-vm:~$ sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.6 (Debian 15.6-1.pgdg120+2))
Type "help" for help.

postgres=# create database otus;

--Создаем таблицу и проверяем наполнение
otus=# create table test_komarov(id int, date_update timestamp);
CREATE TABLE
otus=# insert into test_komarov values(1,now());
INSERT 0 1

otus=# select * from test_komarov;
 id |        date_update
----+----------------------------
  1 | 2024-02-18 15:26:23.967022
(1 row)


komarov@komarov-vm:~$ sudo docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS
        NAMES
d8ef6c44420d   postgres:15   "docker-entrypoint.s…"   14 minutes ago   Up 14 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server

--Останавливаем и удаляем контейнер, проверяем, контенера нет
komarov@komarov-vm:~$ sudo docker stop pg-server
pg-server

komarov@komarov-vm:~$ sudo docker rm pg-server
pg-server

komarov@komarov-vm:~$ sudo docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES

--Подключаем снова:
komarov@komarov-vm:~$ sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15 

komarov@komarov-vm:~$ psql -h localhost -U postgres -d postgres
Password for user postgres:
psql (15.6 (Ubuntu 15.6-1.pgdg20.04+1))
Type "help" for help.

--Проверяем таблицу 
postgres=# \c otus;
You are now connected to database "otus" as user "postgres".
otus=# select*from test_komarov;
 id |        date_update
----+----------------------------
  1 | 2024-02-18 15:26:23.967022
(1 row)

Данные сохранились;

Подключился к контейнеру в котором развернут Postgres c ноутбука из dbeaver 

Выполнил запрос, все работает)




 
