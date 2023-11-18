---
author: pg-x
title: "PostgreSQL 的多进程架构有何优势?"
date: 2023-11-18T10:12:00+08:00
tags: ["postmaster"]
ShowToc: false
TocOpen: false
---

> PostgreSQL 的多进程架构相对于多线程架构有什么优势？如果你认同 "一个查询的后端进程崩溃不会影响其它连接的后端进程，从而提高了系统稳定性" 是其中一个优势，那请继续看下去 🐘。

### postmaster 状态机

`postmaster` 作为 PostgreSQL 的主进程，在启动时负责创建共享内存和信号量。然而，它自身几乎不直接操作这些资源（当然也有例外，如 PMSignalState），这样的设计使得 `postmaster` 逻辑更简单、更可靠。当后端进程崩溃时，`postmaster` 通过重置共享内存和信号量来恢复系统的正常运行。

当 `postmaster` 接收到客户端请求时，它会立即派生（fork）一个子进程。子进程负责对请求进行权限验证，如果验证成功，则成为一个后端进程，这种设计保证了客户端请求的高效处理。

`postmaster` 负责启动一系列子进程，包括辅助（Auxilary）进程和普通（Normal）后端进程，其中 `startup` 进程跟 `postmaster` 紧密合作来保证数据库的一致性，它调用 `StartupXLOG` 执行数据库恢复的流程（本文不展开 recovery 的流程）。

`postmaster` 被实现为一个状态机，父子进程之间通过信号（signal）进行交互，如下是 postmaster 的状态图:

![postmaster state machine](/images/postmaster_state_machine.png)

- postmaster 启动后初始状态为 **PM_INIT**，完成一些初始化工作后，启动 startup 进程，postmaster 进入 **PM_STARTUP** 状态
- 如果 startup 进程完成其工作并正常退出（OS 给 postmaster 发送 SIGCHLD 信号），postmaster 进入 **PM_RUN** 状态
- startup 进程在进行到可以开始 archive recovery 时，给 postmaster 发信号并在共享内存标记 `PMSIGNAL_RECOVERY_STARTED`，postmaster 处理后进入 **PM_RECOVERY** 状态
- startup 持续重放 wal 日志，如果开启了 Hot Standby，当达到一致状态后，会标记 `PMSIGNAL_BEGIN_HOT_STANDBY`，postmaster 进入 **PM_HOT_STANDBY** 状态
- postmaster 在 **PM_RUN** 和 **PM_HOT_STANDBY** 状态都可以接收请求连接，因此这里把两个状态放在了一起
- 当有后端进程异常退出后，postmaster 进入 **PM_WAIT_BACKENDS** 状态，并调用 `HandleChildCrash` 来通知其它进程退出
- 所有进程退出后，postmaster 进入 **PM_NO_CHILDREN** 状态，然后重建共享内存和信号量并启动 startup 进程，postmaster 状态改为 **PM_STARTUP**

可以看出，一个后端进程的异常退出会影响其它子进程。一个简单的验证方法是使用 `kill -9` 去杀死一个查询的后端进程，并查看其它后端进程的 pid 是否改变，参考 [KILL -9 EXPLAINED FOR POSTGRESQL](https://www.cybertec-postgresql.com/en/kill-9-explained-postgresql/)。

社区最近的一个讨论: [How to solve the problem of one backend process crashing and causing other processes to restart?](https://www.postgresql.org/message-id/flat/37b7a067.1773.18bc68459d2.Coremail.yyuansong%40126.com) Tom Lane 的回复:

![](/images/pg_database-wide-restart.png)

### 多进程架构优势

多进程架构确实可以实现子进程之间的相互隔离，确保它们不会相互影响，Nginx 就是一个典型的例子，采用多进程架构来处理并发请求，一个 worker 进程崩溃，其它 worker 进程仍然可以继续运行，保持服务的连续性。

但对于 PostgreSQL 这种使用共享内存来存储关键数据结构和缓存的数据库系统，为了确保数据的一致性，当后端进程异常退出时，需要重建共享内存，并进行崩溃恢复（crash recovery）流程。在恢复过程中，系统通过 WAL 日志文件和其它持久化的数据来还原共享内存中的数据结构，保证在异常发生后数据库能够恢复到一个可用的状态。

回到题目的问题，如果一个查询的后端进程崩溃像多线程模型一样，会导致其它会话被关闭，那 PostgreSQL 的多进程架构还有什么优势？**每个后端进程可单独调试（这点因人而异吧，多线程可以用 thread apply all bt 观察所有线程的状态）**、**更容易控制资源的使用**是我能想到的两点优势。

但反过来看，多线程架构比多进程架构的优势看起来更具吸引力:

- 无需使用共享内存，可使用的内存量没有限制
- 多线程极大地简化了并行算法的实现，线程之间的数据交互和同步可以更加高效地完成
- 线程的上下文切换比起进程更高效，并且线程消耗的内存更少
- 由于所有线程共享相同的内存空间，TLB 的使用效率更高
- The point is that switching to a multi-threaded model makes possible, or at least greatly simplifies, a lot of other development.

PG 社区一直有用多线程替换多进程架构的声音，比如 2017 年 Konstantin Knizhnik 进行过尝试 [Postgres with pthread](https://www.postgresql.org/message-id/flat/9defcb14-a918-13fe-4b80-a0b02ff85527@postgrespro.ru), 今年 6 月 Heikki Linnakangas 在 [Let's make PostgreSQL multi-threaded](https://www.postgresql.org/message-id/flat/31cc6df9-53fe-3cd9-af5b-ac0d801163f4%40iki.fi) 里认为 PG 演进为 single-process & multi-threads 的 idea 相比以往更加成熟了，社区的一些核心成员对此表现出极大的兴趣，Heikki 也将在 PGConf.EU 2023 上对此进行主题演讲: [Multi-threaded PostgreSQL?](https://www.postgresql.eu/events/pgconfeu2023/schedule/session/4845-multi-threaded-postgresql/)。

Hacker News 上相关的讨论: [https://news.ycombinator.com/item?id=36284487](https://news.ycombinator.com/item?id=36284487)

### Takeaways

1. PG 多进程架构做不到子进程之间完全隔离，一个子进程崩溃会导致其它子进程被关闭（SysLogger 是个例外）
2. crash recovery 可能会造成系统短时间不可用（几秒到几分钟都有可能）
3. [不要用 kill -9 去结束会话](https://www.cybertec-postgresql.com/en/cancel-hanging-postgresql-query/)，用 kill /pg_terminate_backend()
4. PG 的多进程架构相对多线程已经没有不可替代的优势了，之所以还未迁移到多线程是因为有些模块的设计留有技术债，改为多线程是一项浩大的工程
