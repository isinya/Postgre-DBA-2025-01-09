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

**Установка PostgreSQL 15 AltLinux**    
   ```sh
apt-get install postgresql15-server postgresql15-server-devel    
/etc/init.d/postgresql initdb    
systemctl enable postgresql.service    
systemctl start postgresql.service
   ```

**Установка PGBench**
   ```sh
apt-get install postgresql15-contrib    
   ```
