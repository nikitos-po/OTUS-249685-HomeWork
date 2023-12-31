# Уровни изолции
## Создать два подключения к Postgres
sudo -u postgres psql

|Connection #1 | Connection #2
|------------------------------|------------------------------
could not change directory to "/root": Отказано в доступе | could not change directory to "/root": Отказано в доступе
psql (15.3 (Debian 15.3-0+deb12u1), server 13.11 (Debian 13.11-0+deb11u1)) | psql (15.3 (Debian 15.3-0+deb12u1), server 13.11 (Debian 13.11-0+deb11u1))
Type "help" for help. | Type "help" for help.
postgres=# | postgres=#

## auto commit
Просмотр текущего состояния autocommit
> \echo :AUTOCOMMIT

|Connection #1 | Connection #2
|--|--
on | on

Отключение AUTOCOMMIT
> \set AUTOCOMMIT OFF

Просмотр текущего состояния autocommit
> \echo :AUTOCOMMIT

|Connection #1 | Connection #2
|--|--
OFF | OFF

## Таблицы и данные

### В первой сессии

Создание отдельной базы для экспериментов
> CREATE DATABASE OTUS

Просмотр баз
> \l

   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------|----------|----------|-------------|-------------|------------|-----------------|-----------------------
 otus      | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            |
 postgres  | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
  | | | | | | | | postgres=CTc/postgres
 template1 | postgres | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            | =c/postgres          +
  | | | | | | | | postgres=CTc/postgres
 zabbix    | zabbix   | UTF8     | ru_RU.UTF-8 | ru_RU.UTF-8 |            | libc            |

Сменим текущую базу на "OTUS"
> \c otus

Создание таблицы.
> create table persons(id serial, first_name text, second_name text);

Проверим что получилось
> otus=*# \dt

List of relations

 Schema |  Name   | Type  |  Owner
--------|---------|-------|----------
 public | persons | table | postgres

Наполним созданную таблицу данными
> insert into persons(first_name, second_name) values('ivan', 'ivanov');
> insert into persons(first_name, second_name) values('petr', 'petrov');
> commit;

## Работа с уровнями изоляции

### read committed

Посмотреть текущий уровень изоляции в обеих сессиях
> show transaction isolation level;

transaction_isolation

|Connection #1 | Connection #2
|----|----
read committed | read committed

#### Сессия #1

Вставка данных
> insert into persons(first_name, second_name) values('sergey', 'sergeev');

#### Сессия #2

Чтение данных
>select * from persons;

 id | first_name | second_name
----|------------|-------------
  1 | ivan       | ivanov
  2 | petr       | petrov

`Данные, вставленные в сессии #1, не видны, т.к. операция вставки не подтверждена командой COMMIT. При текущем уровне изоляции "read committed" это нормальное поведение.`

В транзакции, работающей на уровне _read committed_, запрос SELECT (без предложения FOR UPDATE/SHARE) видит только те данные, которые **были зафиксированы до начала запроса**; он никогда не увидит незафиксированных данных или изменений, внесённых в процессе выполнения запроса параллельными транзакциями.

#### Сессия #1

Подтверждение модификации данных
> commit;

#### Сессия #2

Чтение данных
>select * from persons;

 id | first_name | second_name
----|------------|-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev

`Данные, вставленные в сессии #1, стали видны, т.к. операция вставки теперь подтверждена командой COMMIT. При текущем уровне изоляции "read committed" это нормальное поведение.`

### repeatable read

Посмотреть текущий уровень изоляции в обеих сессиях
> show transaction isolation level;

transaction_isolation

|Connection #1 | Connection #2
|----|----
read committed | read committed

Изменить уровень изоляции
> set transaction isolation level repeatable read;

Посмотреть текущий уровень изоляции в обеих сессиях
> show transaction isolation level;

transaction_isolation
|Connection #1 | Connection #2
|----|----
 repeatable read | repeatable read 

#### Сессия #1

Вставка данных
> insert into persons(first_name, second_name) values('sveta', 'svetova');

#### Сессия #2

Чтение данных
>select * from persons;

 id | first_name | second_name
----|------------|-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev

`Данные, вставленные в сессии #1, не видны, т.к. операция вставки не подтверждена командой COMMIT. При текущем уровне изоляции "repeatable read" это нормальное поведение.`

В режиме _Repeatable Read_ видны только те данные, которые были зафиксированы до начала транзакции, но **не видны** незафиксированные данные и изменения, **произведённые другими транзакциями в процессе выполнения данной транзакции**.

#### Сессия #1

Подтверждение модификации данных
> commit;

#### Сессия #2

Чтение данных
> select * from persons;

 id | first_name | second_name
----|------------|-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev

Завершение текущей сессии
> commit;

Чтение данных
>select * from persons;

 id | first_name | second_name
----|------------|-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova

`Данные, вставленные в сессии #1, стали видны, т.к. сессия завершена командой COMMIT.`

Обе сессии завершены, т.е. мы не ограничены видимостью данных, которые были изменены **до** начала новой сесии. Режим изоляции _repeatable read_ гарантирует повторимость (воспроизводимость) результата выборки внутри открытой сесии.
