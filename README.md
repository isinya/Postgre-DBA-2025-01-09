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


