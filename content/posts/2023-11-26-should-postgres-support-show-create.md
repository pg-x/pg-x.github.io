---
author: pg-x
title: "PostgreSQL 为什么不提供 SHOW CREATE TABLE ？"
date: 2023-11-26T18:48:38+08:00
tags: ["SHOW CREATE TABLE"]
ShowToc: false
TocOpen: false
---

> I know it seems dumb, but postgres really needs to add the simple developer experience stuff like:
>
> SHOW CREATE TABLE;  
> SHOW TABLES;  
> SHOW DATABASES;  
> SHOW PROCESSLIST;
>
> CockroachDB added these aliases ages ago.

如上是 Hacker News 今年 5 月的一个[帖子](https://news.ycombinator.com/item?id=35908991)，相信 PostgreSQL 的用户都知道，**PostgreSQL has its own way!!!**

是的，我们用 `slash commands`，并且热衷这种简洁的方式:

|  MySQL  | PG     |
| --------- | -------- |
| show create table t; | \d+ t |
| show tables; | \d |
| show tables like \"\%test\%\" | \dt public.\*test\* |
| show databases; | \l |
| show processlist; | SELECT * from pg_stat_activity; |

不过 `slash commands` 只能在 psql 中使用，psql 将相应的命令转换成对应的 SQL 查询发送到 server 端处理。大多情况下，这种方式能满足用户的需求，但也有例外，比如:

- 需要返回能够直接创建表的命令而非表结构
- GUI tools / ORM

PG 生态如此强大，当然也有应对的方法，比如用 pg_dump 去获取创建表的语句，也有插件提供 UDF 供应用端调用（当然也可以自己写 UDF 来实现类似的功能）:

- retail_ddl: https://bitbucket.org/adunstan/retailddl
- pgddl: https://github.com/lacanoid/pgddl

但这依然不能覆盖所有的场景，例如，你总不能让 PgAdmin 这样的图形化工具去依赖外部插件提供的 UDF 吧？PgAdmin 同样是在应用端通过查询系统表来将其组装成需要的格式。

### PG 有必要提供 SHOW CREATE TABLE 吗?

大多情况下，psql 的 `slash commands` 已经能够满足人们的需求，加之一些图形化工具有自己 hack 的方式，导致这个特性的优先级相对较低。

不过这个特性其实还是有用的，如果服务端提供了相应的查询语句，客户端就无需自行 hack 来解决问题，所以社区偶尔会出现 SHOW CREATE TABLE 相关的讨论:

- [\describe*](https://www.postgresql.org/message-id/flat/CADkLM%3DeHUZEMi%2BM%3DJjvbdLNBSW6oiSYBpadEq0hvXhtoQd%2Bvfw%40mail.gmail.com)
- [SHOW CREATE](https://www.postgresql.org/message-id/flat/20190705163203.GD24679@fetter.org)
- [Adding SHOW CREATE TABLE](https://www.postgresql.org/message-id/flat/CAFEN2wxsDSSuOvrU03CE33ZphVLqtyh9viPp6huODCDx2UQkYA%40mail.gmail.com)

不过有一些暂未明确的点:

- 在客户端实现，还是在服务端实现？
- 如果在客户端实现，是否考虑将 pg_dump 和 postgres_fdw 中的大量的冗余代码重构到 libpq 或 common 模块
- 如果在 server 端实现，是以 udf 还是 SHOW CREATE TABEL 的方式提供

### 小结

psql、postgres_fdw 以及 pg_dump，都实现了获取表结构的能力，而且它们的实现都是在客户端，为了支持向后兼容，代码中还需要考虑不同的 PG 版本，并且存在大量冗余代码。个人观点:

- SHOW CREATE TABLE 在 PG 中并不是伪需求，提供类似的功能后可以简化工具软件的开发工作
- 支持在 server 侧，且以 UDF 的方式（pg_get_tabledef?）提供该特性
- 如果你只是从 MySQL 数据库转到 PG，没必要等这个特性，建议尽快熟悉 Slash commands
