---
author: pg-x
title: "PostgreSQL 中三个常用的 timeout 参数"
date: 2023-10-19T12:21:22+08:00
tags: []
draft: false
ShowToc: false
TocOpen: false
---

最近在浏览 `guc_table.c` 时注意到 PostgreSQL 有很多与超时相关的参数可供用户设置，在这些参数中，我想介绍三个我认为最常用的参数:

- statement_timeout: Sets the maximum allowed duration of any statement.
- idle_in_transaction_session_timeout: Sets the maximum allowed idle time between queries, when in a transaction.
- idle_session_timeout: Sets the maximum allowed idle time between queries, when not in a transaction.

### statement_timeout

PostgreSQL 中长事务往往会带来一些问题，比如 table bloat（由于旧版本记录不能及时回收）、占用资源（锁、内存）等，因此有些实例会设置一个合理的 statement_timeout 来自动杀死运行时间过长的查询。

```SQL
postgres=# set session statement_timeout = '10s';
SET
postgres=# select pg_sleep(1000);
ERROR:  canceling statement due to statement timeout
postgres=#
```

但这个参数对于 `idle in transaction` 的 session 不起作用，比如:

```SQL
postgres=# set application_name = 'pg-x';
SET
postgres=# set session statement_timeout = '10s';
SET
postgres=# create table t1(id int);
CREATE TABLE
postgres=# insert into t1 values(1),(2);
INSERT 0 2
postgres=# begin;
BEGIN
postgres=*# update t1 SET id=2 where id=1;
UPDATE 1
```

开启事务但一直不 commit，该 session 一直处于 `idle in transaction` 状态，在另一个客户端查询:

```SQL
postgres=# select pid, application_name, xact_start, query_start, state from pg_stat_activity where application_name='pg-x';
  pid   | application_name |          xact_start           |          query_start          |        state
--------+------------------+-------------------------------+-------------------------------+---------------------
 201736 | pg-x             | 2023-10-14 16:09:10.389653+00 | 2023-10-14 16:10:17.354243+00 | idle in transaction
(1 row)
```

### idle_in_transaction_session_timeout

idle_in_transaction_session_timeout 是一个用于设置事务在空闲状态下超时时间的参数，当一个事务处于空闲状态（没有活动查询）并且超过了指定的时间限制时，PostgreSQL 将自动终止该事务并释放相关资源。

```SQL
postgres=# set session idle_in_transaction_session_timeout = '10s';
SET
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
         201736
(1 row)

postgres=# begin;
BEGIN
postgres=*# commit;
FATAL:  terminating connection due to idle-in-transaction timeout
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
         202340
(1 row)

```

可以看到在 `begin` 命令执行 10s 之后再去执行 `commit`，客户端当前连接的 backend 已经被杀死了，并重新连接了另一个 backend。

### idle_session_timeout

idle_session_timeout 是 PG14 引入的一个参数，当 backend 一直没有查询（处于 idle 状态）超过参数设定的时间时，进程会被杀死。

```SQL
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
         202340
(1 row)

postgres=# set session idle_session_timeout = '10s';
SET
postgres=#
postgres=# \d
FATAL:  terminating connection due to idle-session timeout
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
postgres=# select pg_backend_pid();
 pg_backend_pid
----------------
         202453
(1 row)
```

在设置了 idle_session_timeout 之后等待 10s，然后随便执行一个命令，即可验证当前的后端进程已被杀死。

### 小结

本文介绍了三个常用的超时参数，根据实际情况进行合理设置能让你的系统运行地更稳健。另一个 `lock_timeout` 用于设置事务在等待锁资源时的超时时间，当一个事务等待获取锁资源的时间超过指定的时间限制时，PostgreSQL 会自动中断该事务，但这个我用的不多，所以未放在正文中。
