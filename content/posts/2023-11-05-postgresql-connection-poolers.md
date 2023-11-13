---
author: pg-x
title: "PostgreSQL Connection Poolers"
date: 2023-11-05T10:00:56+08:00
tags: ["pgbouncer", "pgpool", "odyssey", "pgcat"]
ShowToc: false
TocOpen: false
---

数据库连接池（Connection Pooler）是一种用于管理和复用数据库连接的组件，它允许应用程序在需要与数据库建立连接时从连接池中获取连接，而非每次都创建新的连接，以减少连接创建和销毁的开销。今天我们介绍 PostgreSQL 数据库生态中常用的四个具有连接池能力的工具: [PgBouncer](https://github.com/pgbouncer/pgbouncer)、[Pgpool-II](https://github.com/pgpool/pgpool2)、[Odyssey](https://github.com/yandex/odyssey)、[PgCat](https://github.com/postgresml/pgcat)。

### 为什么需要连接池?

通常 DBA 会给 PostgreSQL 实例设置一个合理的 `max_connections` 值，当连接数超过这个值后会报 `FATAL: remaining connection slots are reserved for ...` 的错误。

有些人认为 PostgreSQL 不能处理太多连接是因为它的进程模型（为每个连接创建一个 backend 进程）会占用大量内存，但其实更重要的原因是 [Snapshot scalability](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/analyzing-the-limits-of-connection-scalability-in-postgres/ba-p/1757266)，当有大量连接（即使状态为 idle）时，获取快照信息（[`GetSnapshotData`](https://github.com/postgres/postgres/blob/REL_12_STABLE/src/backend/storage/ipc/procarray.c#L1545)）成为了性能瓶颈。

PG 14 针对 Snapshot scalability 的问题进行了优化: [Improving Postgres Connection Scalability: Snapshots](https://techcommunity.microsoft.com/t5/azure-database-for-postgresql/improving-postgres-connection-scalability-snapshots/ba-p/1806462)，降低了 idle 状态连接对数据库性能的影响，因此我们现在可以把 `max_connections` 设置为一个较大的值（maybe 1000~5000?）。那这是否意味着我们不再需要连接池了？答案是否定的，我能想到的两个原因:

1. 应用端大量的短连接频繁创建、删除有性能开销
2. 微服务架构下存在大量应用，相对单体架构需要更多的连接，连接数依然可能耗尽

依据连接池运行的位置可简单将其分为两类，一种是存在于应用程序的 driver 里，如 [pgjdbc-ng](https://github.com/impossibl/pgjdbc-ng) 库提供的 Connection Pool DataSource；另一种是单独部署的进程。后文介绍的四个连接池都归为第二类。

### PgBouncer

PgBouncer 是一款轻量的连接池解决方案，单线程、低开销（单个连接占用约 2kB 内存）、使用 [libevent](https://github.com/libevent/libevent) 事件驱动库来处理异步 I/O，功能简单，配置容易。

三种使用模式:

- 会话模式（Session pooling）: 一个客户连接对应一个服务端连接，支持 PostgreSQL 所有特性，常用于多个数据库的连接映射（Connection Mapping）
- 事务模式（Transaction pooling）: 一个服务器连接只在事务期间分配给客户端。当 PgBouncer 察觉到事务结束时，连接将被放回连接池中。这种模式会破坏几个基于会话的功能
- 语句模式（Statement pooling）: 不允许多语句事务，意味着 autocommit 必须打开

事务模式**不支持**的特性有:

```TXT
- SET/RESET
- LISTEN
- WITH HOLD CURSOR
- PREPARE / DEALLOCATE（不支持直接发送字符串语句，但支持协议层面的 Prepared Statements）
- PRESERVE/DELETE ROWS temp tables
- LOAD statement
- Session-level advisory locks
```
PgBouncer 1.21 版本将事务模式支持了 [Protocol-level Prepared Statements](https://www.crunchydata.com/blog/prepared-statements-in-transaction-mode-for-pgbouncer) 特性。[PR](https://github.com/pgbouncer/pgbouncer/pull/845)

当一个 PgBouncer 处理能力达到上限时，可以部署多个 PgBouncer。也正是由于 PgBouncer 的单线程轻量模型，让它可以在应用程序没有线程池库的情况下部署在近应用端，提供线程池的能力。一个常见的部署架构如下:

![pgbouncer deployment](/images/pgbouncer_with_haproxy.png)

##### *PgBouncer 还可以通过 unix domain socket 将老进程的 fd 迁移到新进程来支持在线重启、在线升级的特性。*

### Pgpool-II

Pgpool-II 是一个位于应用程序和 PostgreSQL 之间的中间件，旨在提供单机 PG 不具备的特性，如:

- 高可用 —— 通过监视后端 PostgreSQL 实例的可用性，在检测到故障时自动切换到其它可用的实例，实现了自动故障检测和故障转移机制，确保数据库的高可用性和持续服务
- 负载均衡（读写分离） —— 将只读查询分发到多个 PostgreSQL 实例以减轻主库负载，进而获得更高的性能

除此基本功能之外，Pgpool-II 还提供一些其它非常实用的特性:

- 连接池: 管理和维护与 PostgreSQL 数据库的连接池，以减少连接创建和关闭带来的开销
- 在线恢复: 将 crash 的旧主变为 standby 重新加入集群
- Watchdog: 为了解决单个 Pgpool-II 造成的单点故障，Watchdog 协调多个 Pgpool-II 节点构成一个集群，有效避免了单点故障或脑裂的问题
- 查询缓存: In Memory Query Cache 保存 SELECT 语句及其结果的对应关系，相同的语句查询会从缓存返回结果

Pgpool-II 是一个多进程模型:

![Process architecture of Pgpool-II](https://www.pgpool.net/docs/44/en/html/process-diagram.gif)

Parent 作为主进程是所有其它进程的父，它负责派生子进程，child 进程接收并处理来自应用端的连接（每个 child 维护一个私有连接池缓存，一个 database/user 对应的多个连接可以由任意 child 进程处理），worker 进程负责检测流复制延迟，pcp 进程用于管理 Pgpool-II 自身状态，watchdog 进程则负责 Pgpool-II 自身的高可用。

Pgpool-II 可同时处理的连接数为 num_init_children (default 32) * max_pool(default 4)，这两个参数的配置需要跟 PostgreSQL 实例的配置相匹配:

```txt
max_pool*num_init_children <= (max_connections - superuser_reserved_connections) (no query canceling needed)
max_pool*num_init_children*2 <= (max_connections - superuser_reserved_connections) (query canceling needed)
```

Pgpool-II 在创建连接的时候会解析服务端发送的 `ParameterStatus`，比如 `in_hot_standby` 可用于判断该实例是否为热备，当解析到应用发来的是只读查询时，可以将该查询分发到 hot standby 节点上。

一个典型的高可用集群架构如下:

![Cluster System Configuration](https://www.pgpool.net/docs/44/en/html/cluster_40.gif)

Pgpool-II 提供了更多的功能，但配置起来也更加复杂，更容易 `shoot yourself in the foot`。由于 Pgpool-II 只有 session 模式，为每个应用端连接维护一个后端连接，它作为 connection pooler 并不能处理并发极高的场景。

[Pgpool-II - WHAT, WHY, AND WHERE?](https://www.youtube.com/watch?v=aJd4ICzEYHI) 👈🏻 这个视频介绍了 Pgpool-II 最适合的使用场景，也指出了如果只是需要 Connection pooler，Pgpool-II 不是最佳的选择。

### Odyssey

Odyssey 是一个由 Yandex 开发的高性能、高可靠的 PostgreSQL 连接池。Odyssey 使用了多线程模型，其比较新颖的地方是它开发了一个 [Machinarium](https://github.com/yandex/odyssey/tree/master/third_party/machinarium) 库，可以用同步、过程化的编程方式来编写异步事件驱动程序。

与 PgBouncer 类似，Odyssey 也提供了[三种工作模式](https://cloud.yandex.com/en/docs/managed-postgresql/concepts/pooling):

- Session mode (default): 连接在客户端首次查询数据库时建立，并在客户端终止会话之前保持连接
- Transaction mode: 连接在客户端首次查询数据库时建立，并在事务结束时断开连接
- Query mode: 不允许多语句事务，autocommit 必须打开

同样，在事务模式下 Odyssey 也有不支持的特性:

```TXT
- Temporary tables
- cursors
- advisory locks
- PREPARE
```

支持 [Protocal level Prepared Statement](https://www.youtube.com/watch?v=xlDIqTW079s)，且早于 PgBouncer。

与 PgBouncer 不同的是，Odyssey 的多线程模式在高负载情况下能够处理更多的请求。同时 Odyssey 在事务结束之后尽可能让来自同一个客户端的事务复用同一个连接。

Odyssey 进程包含如下几个核心模块:

```
                                    main()
                                .----------.
                                | instance |
                thread          '----------'
                .--------.                          .-------------.
                | system |                          | worker_pool |
                '--------'                          '-------------'
        .--------.    .---------.           .---------.         .---------.
        | router |    | servers |           | worker0 |   ...   | workerN |
        '--------'    '---------'           '---------'         '---------'
        .---------.    .------.               thread              thread
        | console |    | cron |
        '---------'    '------'
```

各模块的功能可参考 [Odyssey architecture and internals](https://github.com/yandex/odyssey/blob/master/documentation/internals.md) 和作者在 CMU ¡Databases! 的介绍 [Odyssey: PostgreSQL Connection Proxy!](https://www.youtube.com/watch?v=VEYdZL0bU-I&t=1928s)。

### PgCat

PgCat 是 [PostgresML](https://postgresml.org/) 开发的一款连接池工具，使用 Rust 语言编写，旨在[将 PgBouncer 提升到一个新的水平](https://news.ycombinator.com/item?id=30267539)。PgCat 提供和 PgBouncer 一样的 Session mode 和 Transaction mode，多线程模型，使用 [Tokio](https://github.com/tokio-rs/tokio) 异步框架处理请求，能够充分利用多核系统的资源。事务模式同样支持 [Protocal level Prepared Statement](https://postgresml.org/blog/making-postgres-30-percent-faster-in-production)。

PgCat 支持的特性:

- Query load balancing: 使用可配置的负载均衡策略，将查询均匀地分发到所有可用的副本中
- High availability: PgCat 维护了一个内部映射，用于记录健康和不健康的副本，并且只将流量路由到健康的副本上。这种机制确保只有可靠的副本参与查询处理，从而提高系统的可靠性
- Read/write query separation: 通过解析 SQL 以确定查询的读写意图，并根据需要将查询路由到主库或热备库
- Sharding: 根据配置配置将查询路由到不同的节点，需要跟其它 shard 工具一起使用

有一些公司已经将 PgCat 用在生产环境，比如 PostgreML 和 [Instacart](https://tech.instacart.com/adopting-pgcat-a-nextgen-postgres-proxy-3cf284e68c2f)。

### 对比

下表给出了四种工具在连接池特性上的一些对比:

|           | PgBouncer | Pgpool-II | Odyssey | PgCat |
| --------- | --------- | --------- | ------- | ----- |
| 线程模型   |  单线程 | 多进程      | 多线程 | 多线程 |
| 异步框架   | libevent |    | Machinarium(epoll) | tokio |
| load balance | 解析同一个域名下的多个 副本来支持负载均衡 | ✅ | ❌ | ✅ |
| read/write separation | ❌ | ✅ | ❌ | ✅ |
| Pooling modes | session mode, transaction mode, statement mode | session mode only | session mode, transaction mode, query mode | session mode, transaction mode |
| 单点故障 | 存在 | 可配置 watchdog 消除单点故障 | 存在 | 存在 |


### 总结

本文介绍了 PostgreSQL 生态常用的四个连接池，其中 PgBouncer 和 Pgpool-II 是两个成熟的工具，PgBouncer 被很多人认为是连接池的事实标准(de facto standard)，Pgpool-II 提供了很多额外的特性，反而作为连接池其性能有点差强人意（不支持 transaction mode）。Odyssey、PgCat 可以称作 next generation PostgreSQL pooler 或 modern PostgreSQL pooler，利用多线程和异步事件编程充分发挥计算机的资源，解决 PgBouncer 性能上的不足，并提供一些额外特性。

就我个人而言，我对 Odyssey 的 Machinarium 库和 PgCat 的读写分离非常感兴趣，期待在下一个项目上尽快尝试一下 🧐。

##### *还有一个项目 [pgagroal](https://github.com/agroal/pgagroal) 也是一个比较新的 connection pooler，感兴趣可自行研究。*

