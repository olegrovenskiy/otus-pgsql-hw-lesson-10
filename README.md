# ДЗ otus-pgsql-hw-lesson-10

# 1. Задание 1

###  Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.

    
    postgres=# SHOW log_lock_waits;
     log_lock_waits
    ----------------
     off
    (1 row)

включаем логирование


    postgres=# ALTER SYSTEM SET log_lock_waits = on;
    ALTER SYSTEM
    postgres=#
    
    postgres=# SELECT pg_reload_conf();
     pg_reload_conf
    ----------------
     t
    (1 row)
    
    postgres=# SHOW log_lock_waits;
     log_lock_waits
    ----------------
     on
    (1 row)

также установил таймер на 200 мс

    postgres=# SET lock_timeout = '200ms';
    SET
    postgres=#
    postgres=#
    postgres=# SHOW lock_timeout;
     lock_timeout
    --------------
     200ms
    (1 row)


Сессия 1

        postgres=# begin;
        BEGIN
        postgres=*# update test_text set t = 'test1' where t = 'test11';
        UPDATE 1
        postgres=*# 

Сессия 2

    postgres=# begin
    postgres-# ;
    BEGIN
    Time: 0.190 ms
    postgres=*# update test_text set t = 'test21' where t = 'test11';
    ERROR:  canceling statement due to lock timeout
    CONTEXT:  while updating tuple (0,5) in relation "test_text"
    Time: 200.782 ms
    postgres=!#


Информация о данном событии есть в лог файле

    2024-01-18 02:22:12.316 EST [97057] ERROR:  canceling statement due to lock time                                                                                                                                                             out
    2024-01-18 02:22:12.316 EST [97057] CONTEXT:  while updating tuple (0,5) in rela                                                                                                                                                             tion "test_text"
    2024-01-18 02:22:12.316 EST [97057] STATEMENT:  update test_text set t = 'test21                                                                                                                                                             ' where t = 'test11';
    2024-01-18 02:22:34.419 EST [92566] LOG:  checkpoint starting: time



# 2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.


# 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?


# 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
