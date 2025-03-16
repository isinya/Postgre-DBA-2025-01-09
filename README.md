# Postgre-DBA-2025-01 Занятие #09
Блокировки

**Домашнее задание**

> 1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.

Текущие настройки:
   ```sql
postgres=# SHOW log_lock_waits;
 log_lock_waits
----------------
 off
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 1s
   ```
Новые настройки
   ```sql
postgres=# ALTER SYSTEM SET log_lock_waits = on;
ALTER SYSTEM
postgres=# ALTER SYSTEM SET deadlock_timeout = 200;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
postgres=# SHOW log_lock_waits; 
 log_lock_waits
----------------
 on
postgres=# SHOW deadlock_timeout;
 deadlock_timeout
------------------
 200ms
   ```
> Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
   ```sql
postgres=# begin; /* seesion #01 */
BEGIN
postgres=*# select * From accounts Where acc_no = 1 for update;
 acc_no | amount
--------+---------
      1 | 1100.00
postgres=# Select pg_sleep(1);
 pg_sleep
...
postgres=*# commit;
COMMIT

postgres=# begin; /* seesion #02 */
BEGIN
postgres=*# update accounts set amount = amount + 100 Where acc_no = 1;
UPDATE 1
postgres=*# commit;
COMMIT
   ```

Логгирование блокировок длительностью более **200** мс. Блокировка предоставлена через **9435.971 мс**
   ```sh
[root@altLinux01 ~]# cat /var/lib/pgsql/data/log/postgresql-2025-03-16_181752.log
2025-03-16 18:42:36.091 MSK [10456] СООБЩЕНИЕ:  процесс 10456 продолжает ожидать в режиме ShareLock блокировку "транзакция 755" в течение 200.701 мс
2025-03-16 18:42:36.091 MSK [10456] ПОДРОБНОСТИ:  Process holding the lock: 10332. Wait queue: 10456.
2025-03-16 18:42:36.091 MSK [10456] КОНТЕКСТ:  при изменении кортежа (0,5) в отношении "accounts"
2025-03-16 18:42:36.091 MSK [10456] ОПЕРАТОР:  update accounts set amount = amount + 100 Where acc_no = 1;
2025-03-16 18:42:45.327 MSK [10456] СООБЩЕНИЕ:  процесс 10456 получил в режиме ShareLock блокировку "транзакция 755" через 9435.971 мс
2025-03-16 18:42:45.327 MSK [10456] КОНТЕКСТ:  при изменении кортежа (0,5) в отношении "accounts"
2025-03-16 18:42:45.327 MSK [10456] ОПЕРАТОР:  update accounts set amount = amount + 100 Where acc_no = 1;
   ```
> 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.
> Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны.
> Пришлите список блокировок и объясните, что значит каждая.

Сессия #01
   ```sql
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          783 |          10915
(1 строка)

postgres=*#
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
UPDATE 1
postgres=*# SELECT * FROM locks_v WHERE pid = 10915;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 10915 | relation      | accounts | RowExclusiveLock | t
 10915 | transactionid | 783      | ExclusiveLock    | t
   ```
Транзакция #1 (10915) Получена экслюзивная блокировка собственного номера транзакции и блокировка таблицы в режиме RowExclusiveLock.


Сессия #02
   ```sql
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          784 |          10967
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
/* транзакци повисла */
postgres=*# SELECT * FROM locks_v WHERE pid = 10967;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 10967 | relation      | accounts   | RowExclusiveLock | t
 10967 | transactionid | 784        | ExclusiveLock    | t
 10967 | transactionid | 783        | ShareLock        | f
 10967 | tuple         | accounts:1 | ExclusiveLock    | t
  ```
Транзакция #2 (10967) Получена экслюзивная блокировка собственного номера транзакции, блокировка таблицы в режиме RowExclusiveLock, блокировка версии строки.
Блокировка номера транзакции уже обновляющей данную строку (acc_no = 1) не предоставлена - поэтому транзакция повисла.

Сессия #03
   ```sql
postgres=# BEGIN;
BEGIN
postgres=*# SELECT txid_current(), pg_backend_pid();
 txid_current | pg_backend_pid
--------------+----------------
          785 |          11178
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
/* транзакци повисла */
postgres=*# SELECT * FROM locks_v WHERE pid = 11178;
  pid  |   locktype    |   lockid   |       mode       | granted
-------+---------------+------------+------------------+---------
 11178 | relation      | accounts   | RowExclusiveLock | t
 11178 | transactionid | 785        | ExclusiveLock    | t
 11178 | tuple         | accounts:1 | ExclusiveLock    | f
  ```
Транзакция #3 (11178) Получена экслюзивная блокировка собственного номера транзакции, блокировка таблицы в режиме RowExclusiveLock.
Блокировка блокировка версии строки не предоставлена (заблокирована в транзакции #3) - поэтому транзакция повисла.


Сессия #01
   ```sql
postgres=*# commit;
COMMIT
postgres=# SELECT * FROM locks_v WHERE pid = 10915;
 pid | locktype | lockid | mode | granted
-----+----------+--------+------+---------
(0 строк)
  ```

Транзакция #1 (10915). Выполнена модификация строки. Все заблокированные ресурсы освобождены.
Сессия #01
   ```sql
postgres=# SELECT * FROM locks_v WHERE pid = 10967;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 10967 | relation      | accounts | RowExclusiveLock | t
 10967 | transactionid | 784      | ExclusiveLock    | t
 ```

Транзакция #2 (10967). Уровень изоляции ReadCommited - Сейчас транзакция просто изменяет данные (изменения транзакции #1 зафиксированы). 
Получена экслюзивная блокировка собственного номера транзакции, блокировка таблицы в режиме RowExclusiveLock.

Сессия #01
   ```sql
postgres=# SELECT * FROM locks_v WHERE pid = 11178;
  pid  |   locktype    |  lockid  |       mode       | granted
-------+---------------+----------+------------------+---------
 11178 | relation      | accounts | RowExclusiveLock | t
 11178 | transactionid | 784      | ShareLock        | f
 11178 | transactionid | 785      | ExclusiveLock    | t
 ```

Транзакция #3 (11178) ожидает завершения транзакции #2, не пытается получить блокировку версии строки. Такова логика работы Postgres.

COMMIT в сессии #02 и сессии #03 фиксирует изменение данных. Освобождает ресурсы. 
>Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

Сессия #01
   ```sql
postgres=# begin;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;
UPDATE 1
postgres=*# UPDATE accounts SET amount = amount + 5 WHERE acc_no = 3;
UPDATE 1
   ```
Сессия #02
   ```sql
postgres=# begin;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 10.00 WHERE acc_no = 2;
UPDATE 1
postgres=*# UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
   ```
Сессия #03
   ```sql
postgres=# begin;
BEGIN
postgres=*# UPDATE accounts SET amount = amount - 5.00 WHERE acc_no = 3;
UPDATE 1
postgres=*# UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
ОШИБКА:  обнаружена взаимоблокировка
ПОДРОБНОСТИ:  Процесс 11178 ожидает в режиме ShareLock блокировку "транзакция 787"; заблокирован процессом 10967.
Процесс 10967 ожидает в режиме ShareLock блокировку "транзакция 786"; заблокирован процессом 10915.
Процесс 10915 ожидает в режиме ShareLock блокировку "транзакция 788"; заблокирован процессом 11178.
ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
КОНТЕКСТ:  при изменении кортежа (0,2) в отношении "accounts"
   ```

Те же PID-ы и действия вызвавшие блокировку видны в логе Postgres.
   ```sh
2025-03-16 21:18:36.521 MSK [11178] СООБЩЕНИЕ:  процесс 11178 обнаружил взаимоблокировку, ожидая в режиме ShareLock блокировку "транзакция 787" в течение 200.304 мс
2025-03-16 21:18:36.521 MSK [11178] ПОДРОБНОСТИ:  Process holding the lock: 10967. Wait queue: .
2025-03-16 21:18:36.521 MSK [11178] КОНТЕКСТ:  при изменении кортежа (0,2) в отношении "accounts"
2025-03-16 21:18:36.521 MSK [11178] ОПЕРАТОР:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
2025-03-16 21:18:36.522 MSK [11178] ОШИБКА:  обнаружена взаимоблокировка
2025-03-16 21:18:36.522 MSK [11178] ПОДРОБНОСТИ:  Процесс 11178 ожидает в режиме ShareLock блокировку "транзакция 787"; заблокирован процессом 10967.
        Процесс 10967 ожидает в режиме ShareLock блокировку "транзакция 786"; заблокирован процессом 10915.
        Процесс 10915 ожидает в режиме ShareLock блокировку "транзакция 788"; заблокирован процессом 11178.
        Процесс 11178: UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
        Процесс 10967: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 1;
        Процесс 10915: UPDATE accounts SET amount = amount + 5 WHERE acc_no = 3;
2025-03-16 21:18:36.522 MSK [11178] ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2025-03-16 21:18:36.522 MSK [11178] КОНТЕКСТ:  при изменении кортежа (0,2) в отношении "accounts"
2025-03-16 21:18:36.522 MSK [11178] ОПЕРАТОР:  UPDATE accounts SET amount = amount + 10.00 WHERE acc_no = 2;
2025-03-16 21:18:36.522 MSK [10915] СООБЩЕНИЕ:  процесс 10915 получил в режиме ShareLock блокировку "транзакция 788" через 65585.346 мс
2025-03-16 21:18:36.522 MSK [10915] КОНТЕКСТ:  при изменении кортежа (0,3) в отношении "accounts"
2025-03-16 21:18:36.522 MSK [10915] ОПЕРАТОР:  UPDATE accounts SET amount = amount + 5 WHERE acc_no = 3;
   ```

