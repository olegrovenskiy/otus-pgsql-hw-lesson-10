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



# 2. Задание 2.

### Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.

будем обновлять строку в таблице test_txt

        postgres=# select * from test_text;
           t
        -------
         test2
         test3
         test1
        (3 rows)

командами

begin;
update test_text set t = 'testNEW1/2/3' where t = 'test1';

Сессия 1

        postgres=# SELECT pg_backend_pid();
         pg_backend_pid
        ----------------
                  97057
        (1 row)

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 97057;

        postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        postgres-*# FROM pg_locks WHERE pid = 97057;
           locktype    | relation  | virtxid |   xid   |       mode       | granted
        ---------------+-----------+---------+---------+------------------+---------
         relation      | pg_locks  |         |         | AccessShareLock  | t
         relation      | test_text |         |         | AccessShareLock  | t
         relation      | test_text |         |         | RowExclusiveLock | t
         virtualxid    |           | 4/5509  |         | ExclusiveLock    | t
         transactionid |           |         | 1600550 | ExclusiveLock    | t
        (5 rows)
        
        Time: 0.550 ms
        postgres=*#

locktype

Тип блокируемого объекта: relation (отношение), extend (расширение отношения), frozenid (замороженный идентификатор), page (страница), tuple (кортеж), transactionid (идентификатор транзакции), virtualxid (виртуальный идентификатор), spectoken (спекулятивный маркер), object (объект), userlock (пользовательская блокировка) или advisory (рекомендательная)

В сессии 1 блокировки таблицы и транзакции, все в статусе granted, всё предоставлено


Сессия 2

        postgres=# SELECT pg_backend_pid();
         pg_backend_pid
        ----------------
                  97054

SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 97054;

transactionid xid Идентификатор транзакции, являющийся целью блокировки, либо NULL, если цель блокировки — другой объект
Признак granted устанавливается в строке, представляющей блокировку, удерживаемую указанным процессом. 

        postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        postgres-*# FROM pg_locks WHERE pid = 97054;
           locktype    | relation  | virtxid |   xid   |       mode       | granted
        ---------------+-----------+---------+---------+------------------+---------
         relation      | test_text |         |         | RowExclusiveLock | t
         virtualxid    |           | 3/88    |         | ExclusiveLock    | t
         transactionid |           |         | 1600551 | ExclusiveLock    | t
         transactionid |           |         | 1600550 | ShareLock        | f
         tuple         | test_text |         |         | ExclusiveLock    | t
        (5 rows)
        
        Time: 0.458 ms
        postgres=*#

Во 2ой сессии блокировка таблицы и транзакции и к ряду в таблице

транзакция пытается получить блокировку таблицы в режиме ShareLock, но не может, видно из строки :
transactionid |           |         | 1600550 | ShareLock        | false -- означает что другой процесс удерживает

мы также можем увидеть информацию какой процесс блокирует

        postgres=*# SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(97054)) \gx
        -[ RECORD 1 ]----+-------------------------------------------------------------------------
        datid            | 5
        datname          | postgres
        pid              | 97057
        leader_pid       |
        usesysid         | 10
        usename          | postgres
        application_name | psql
        client_addr      |
        client_hostname  |
        client_port      | -1
        backend_start    | 2024-01-18 02:10:59.779149-05
        xact_start       | 2024-01-18 04:21:36.407961-05
        query_start      | 2024-01-18 07:06:15.841288-05
        state_change     | 2024-01-18 07:06:15.84129-05
        wait_event_type  |
        wait_event       |
        state            | active
        backend_xid      | 1600550
        backend_xmin     | 1600550
        query_id         |
        query            | SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(97057))
        backend_type     | client backend
        
        Time: 1.342 ms
        postgres=*#


Сессия 3


        postgres=# SELECT pg_backend_pid();
         pg_backend_pid
        ----------------
                  97393
        (1 row)
        
        postgres=#


SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
FROM pg_locks WHERE pid = 97393;




        postgres=*# SELECT locktype, relation::REGCLASS, virtualxid AS virtxid, transactionid AS xid, mode, granted
        postgres-*# FROM pg_locks WHERE pid = 97393;
           locktype    | relation  | virtxid |   xid   |       mode       | granted
        ---------------+-----------+---------+---------+------------------+---------
         relation      | test_text |         |         | RowExclusiveLock | t
         virtualxid    |           | 5/329   |         | ExclusiveLock    | t
         transactionid |           |         | 1600552 | ExclusiveLock    | t
         tuple         | test_text |         |         | ExclusiveLock    | f
        (4 rows)
        
        Time: 0.462 ms
        postgres=*#

        мы ожидаем блокировки данных - 
        tuple         | test_text |         |         | ExclusiveLock    | false -- но данные в отношении удерживаются другим процессом 

    получим информацию кто нас блокирует

        postgres=*# SELECT * FROM pg_stat_activity WHERE pid = ANY(pg_blocking_pids(97393)) \gx
        -[ RECORD 1 ]----+---------------------------------------------------------
        datid            | 5
        datname          | postgres
        pid              | 97054
        leader_pid       |
        usesysid         | 10
        usename          | postgres
        application_name | psql
        client_addr      |
        client_hostname  |
        client_port      | -1
        backend_start    | 2024-01-18 02:10:49.316687-05
        xact_start       | 2024-01-18 04:22:12.305248-05
        query_start      | 2024-01-18 04:22:41.096809-05
        state_change     | 2024-01-18 04:22:41.096812-05
        wait_event_type  | Lock
        wait_event       | transactionid
        state            | active
        backend_xid      | 1600551
        backend_xmin     | 1600550
        query_id         |
        query            | update test_text set t = 'test222' where t = 'testNEW1';
        backend_type     | client backend
        
        Time: 1.078 ms
        postgres=*#



в лог файле:

        2024-01-18 04:22:42.097 EST [97054] LOG:  process 97054 still waiting for ShareLock on transaction 1600550 after 1000.134 ms
        2024-01-18 04:22:42.097 EST [97054] DETAIL:  Process holding the lock: 97057. Wait queue: 97054.
        2024-01-18 04:22:42.097 EST [97054] CONTEXT:  while updating tuple (0,11) in relation "test_text"
        2024-01-18 04:22:42.097 EST [97054] STATEMENT:  update test_text set t = 'test222' where t = 'testNEW1';
        2024-01-18 04:23:20.424 EST [97393] LOG:  process 97393 still waiting for ExclusiveLock on tuple (0,11) of relation 16622 of database 5 after 1000.154 ms
        2024-01-18 04:23:20.424 EST [97393] DETAIL:  Process holding the lock: 97054. Wait queue: 97393.
        2024-01-18 04:23:20.424 EST [97393] STATEMENT:  update test_text set t = 'test333' where t = 'testNEW1';
        [root@mck-network-test log]# ^C
        [root@mck-network-test log]#

 По логу мы видим какой процесс ожидает какого, тип блокировки и какой запрос блокирует









# 3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?


# 4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?
