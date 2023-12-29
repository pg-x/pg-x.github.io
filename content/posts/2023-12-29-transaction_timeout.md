---
author: pg-x
title: "PostgreSQL 有必要增加一个 transaction_timeout 参数吗？"
date: 2023-12-29T19:00:30+08:00
tags: []
ShowToc: false
TocOpen: false
---

PostgreSQL 中常用的三个 timeout 参数有: statement_timeout、idle_in_transaction_session_timeout 和 idle_session_timeout，它们控制的时间范围如下图所示:

![timeouts](/images/three_timeout.png)

- statement_timeout 用于限制单个 query 执行的时间
- idle_in_transaction_session_timeout 用于限制在事务中闲置的时长
- idle_session_timeout 用于限制连接处于空间的时长

前两个参数很早之前就存在了，idle_session_timeout 则是由 PG 14 引入。

PG 14 之前，过多的连接会导致 [Snapshot scalability](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/analyzing-the-limits-of-connection-scalability-in-postgres/ba-p/1757266) 的问题，当然这包含 idle 连接。PG 14 对此进行了优化，处于 idle 状态的连接不会对性能有太多的影响。

但即使不影响性能，在 pg_stat_activity 视图中看到太多的空闲连接也足够让人恼火，于是中国的 PG 贡献者 Japin Li 提议增加一个 idle_session_timeout，让 DBA 自己控制是否需要自动关闭超时的空闲连接，于是在 PG 14 引入了这个 GUC。

但这三个参数控制不了事务的执行时长，比如下面的查询:

```SQL
postgres=# begin; select pg_sleep(1); select pg_sleep(1); select pg_sleep(1); select pg_sleep(1); commit;
BEGIN
 pg_sleep 
----------
 
(1 row)

 pg_sleep 
----------
 
(1 row)

 pg_sleep 
----------
 
(1 row)

 pg_sleep 
----------
 
(1 row)

COMMIT
```

2022 年末，Andrey Borodin（Yandex Cloud RDS 负责人）发起了一个讨论：增加一个 [transaction_timeout](https://www.postgresql.org/message-id/flat/CAAhFRxiQsRs2Eq5kCo9nXE3HTugsAAJdSQSmxncivebAxdmBjQ%40mail.gmail.com) 参数来限制单个事务的执行时长，设置该参数后的执行效果为:

```SQL
postgres=# set transaction_timeout to '2s';
SET
postgres=# begin; select pg_sleep(1); select pg_sleep(1); select pg_sleep(1); select pg_sleep(1); commit;
BEGIN
 pg_sleep 
----------
 
(1 row)

FATAL:  terminating connection due to transaction timeout
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Succeeded.
 pg_sleep 
----------
 
(1 row)

 pg_sleep 
----------
 
(1 row)

WARNING:  there is no transaction in progress
COMMIT

```

仔细看上面的图示，你会发现 transaction_timeout 和下面两个参数控制的时间范围有重合:

- statement_timeout
- idle_in_transaction_session_timeout

因此在代码实现上判断了如果 transaction_timeout 设置的值**不大于** statement_timeout、idle_in_transaction_session_timeout 设置的值，可以免去对后二者设置的计时器以减少性能损耗。

该特性目前依然在讨论一些实现细节，是否合入也是未知数。不过我个人认为 transaction_timeout 可能会误杀一些合理事务，不知道您认为这是否是一个有用的特性呢？
