---
author: pg-x
title: "有多少种方法可以判断 PostgreSQL 实例的主备类型？"
date: 2023-10-09T21:56:49+08:00
tags: []
draft: false
ShowToc: false
TocOpen: false
---

在一个典型的 PostgreSQL master-slave 部署中，如何能快速识别谁是主谁是备？本文介绍几种常见的方法。

这些方法可以粗略归为两类:

- 通过客户端连接到实例上
- 能够登录到实例运行的主机上

先看第一类:

### 1.1 pg_is_in_recovery()

pg_is_in_recovery 是一个 PostgreSQL 内置的 UDF，用于确定当前数据库实例是否处于恢复模式。在 PostgreSQL 中，恢复模式是指数据库实例正在进行故障恢复或流复制时的工作状态。

既然能连到实例去执行 SQL，证明肯定不处于故障恢复状态，因此，如果查询返回 true，则表示数据库实例处于恢复模式，如果是 false，则表示数据库实例是主服务器。

```SQL
postgres=# select pg_is_in_recovery();
 pg_is_in_recovery
-------------------
 t
(1 row)
```

当然，如果备机压根没有开启 hot_standby，通过连接时的报错即可知道它为备机:

```SHELL
➜ psql -p 5433 postgres
psql: error: connection to server on socket "/tmp/.s.PGSQL.5433" failed: FATAL:  the database system is not accepting connections
DETAIL:  Hot standby mode is disabled.
```

### 1.2 in_hot_standby

in_hot_standby 是 pg14 引入的一个 INTERNAL 参数（不能被用户更改，只能由进程内部逻辑进行设置）。当实例处于 hot_standby 时该参数查询为 **on**，当实例被 promote 后，参数值为 **off**。

> [This information can also be retrieved via the function pg_is_in_recovery(); the primary use-case for in_hot_standby is that it is automatically reported to clients, who can retrieve the current value via the libpq function PQparameterStatus() without having to execute a query.](https://pgpedia.info/i/in_hot_standby.html)

```SQL
postgres=# show in_hot_standby;
 in_hot_standby
----------------
 on
(1 row)

postgres=# SELECT pg_promote();
 pg_promote
------------
 t
(1 row)

postgres=# show in_hot_standby;
 in_hot_standby
----------------
 off
(1 row)
```

### 1.3 pg_stat_replication

pg_stat_replication 是 PostgreSQL 的系统视图之一，用于提供有关流复制的信息和统计数据。通过分析 pg_stat_replication 视图的输出，可以了解主备服务器之间数据同步的情况。所以如果查到该视图有记录，则表明此实例为主服务器。

```SQL
postgres=# select * from pg_stat_replication \gx
-[ RECORD 1 ]----+------------------------------
pid              | 84891
usesysid         | 10
usename          | zhjwpku
application_name | walreceiver
client_addr      |
client_hostname  |
client_port      | -1
backend_start    | 2023-10-11 19:19:37.613731+08
backend_xmin     |
state            | streaming
sent_lsn         | 0/5000148
write_lsn        | 0/5000148
flush_lsn        | 0/5000148
replay_lsn       | 0/5000148
write_lag        |
flush_lag        |
replay_lag       |
sync_priority    | 0
sync_state       | async
reply_time       | 2023-10-11 19:24:58.047233+08

```

### 1.4 pg_stat_wal_receiver

在 PostgreSQL 中，WAL receiver 进程是在流复制环境中运行的特殊进程，负责接收主服务器发送的事务日志（WAL）并将其写入备份服务器的本地磁盘。通过查询 pg_stat_wal_receiver 视图，可以监控 WAL receiver 的状态及性能指标，了解备份服务器与主服务器之间的数据同步情况，以及复制进程的延迟。如果查到该视图有记录，则表明此实例为备机。

```SQL
postgres=# select * from pg_stat_wal_receiver \gx
-[ RECORD 1 ]---------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
pid                   | 84890
status                | streaming
receive_start_lsn     | 0/5000000
receive_start_tli     | 1
written_lsn           | 0/5000060
flushed_lsn           | 0/5000060
received_tli          | 1
last_msg_send_time    | 2023-10-11 19:22:37.622711+08
last_msg_receipt_time | 2023-10-11 19:22:37.622731+08
latest_end_lsn        | 0/5000060
latest_end_time       | 2023-10-11 19:19:37.615401+08
slot_name             |
sender_host           | /tmp
sender_port           | 5432
conninfo              | user=zhjwpku passfile=/Users/zhjwpku/.pgpass channel_binding=disable dbname=replication port=5432 fallback_application_name=walreceiver sslmode=disable sslcompression=0 sslcertmode=disable sslsni=1 ssl_min_protocol_version=TLSv1.2 gssencmode=disable krbsrvname=postgres gssdelegation=0 target_session_attrs=any load_balance_hosts=disable

```

<br>

再看第二类，登录到主机上，进入 data_directory。

### 2.1 recovery.conf

在较老的 PostgreSQL 版本中，recovery.conf 是一个配置文件，用于配置 PostgreSQL 数据库的流复制。从 PostgreSQL 12 版本开始，recovery.conf 被弃用，取而代之的是使用 standby.signal 文件和 primary_conninfo 参数来配置流复制。所以如果数据目录有 `recovery.conf` 或 `standby.signal` 文件，即可判断该实例为备机。

### 2.2 Data Directory Lockfile

Data Directory Lockfile（数据目录锁文件）是一个用于控制对数据库数据目录的访问的文件 —— postmaster.pid。当 PostgreSQL 启动时，会创建 postmaster.pid 文件并将其放置在数据目录中，该文件包含了 Postmaster 进程的 PID，以及其它与服务器运行状态相关的信息:

```
/*
 * As of Postgres 10, the contents of the data-directory lock file are:
 *
 * line #
 *		1	postmaster PID (or negative of a standalone backend's PID)
 *		2	data directory path
 *		3	postmaster start timestamp (time_t representation)
 *		4	port number
 *		5	first Unix socket directory path (empty if none)
 *		6	first listen_address (IP address or "*"; empty if no TCP port)
 *		7	shared memory key (empty on Windows)
 *		8	postmaster status (see values below)

 ...

/*
 * The PM_STATUS line may contain one of these values.  All these strings
 * must be the same length, per comments for AddToDataDirLockFile().
 * We pad with spaces as needed to make that true.
 */
#define PM_STATUS_STARTING		"starting"	/* still starting up */
#define PM_STATUS_STOPPING		"stopping"	/* in shutdown sequence */
#define PM_STATUS_READY			"ready   "	/* ready for connections */
#define PM_STATUS_STANDBY		"standby "	/* up, won't accept connections */
```

因此，可以通过查看 postmaster.pid 文件中第八行 Postmaster 的状态值来判断 PostgreSQL 实例是否为备机:

```shell
➜ cat postmaster.pid
63383
/Users/zhjwpku/fake_root/pgdata_backup
1696991059
5433
/tmp
localhost
 12698206   1114115
standby
```

但需要注意的是，当备机启用了 hot_standby 特性之后（默认是开启的），该方法不再适用（此时备机的状态为 ready）。

### 2.3 用 Debugger 来打印进程中的全局变量

当然，这并不是回答该问题最好的方法，它需要使用者对源码有一定的了解，不过这种方式也许能解决你的其它问题。

*gdb*
```shell
gdb -batch -ex "set pagination 0" -ex "p in_hot_standby_guc" -p 63383
```

*lldb*
```shell
lldb -p 63383 -b -o "p in_hot_standby_guc"
```

### 小结

本文列举了几种常见的用于区分主备机的方法，相信还有更多区分主备机的方法等着我们一起来发现。
