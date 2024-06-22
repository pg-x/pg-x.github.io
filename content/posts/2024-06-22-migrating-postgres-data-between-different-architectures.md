---
author: pg-x
title: "pg_basebackup 能否用于异构平台数据同步？"
date: 2024-06-22T07:33:22+08:00
tags: ["migrating", "replication", "pg_basebackup"]
ShowToc: false
TocOpen: false
---

Postgres 在 x86 架构下的实例是否可以通过 `pg_basebackup` 同步到 arm 架构？

在一个群里看到一个问题:

![](/images/2024/IMG_8557.jpg)

`pg_basebackup` 是对 Postgres 实例数据文件在目的端的一个精确拷贝:

> pg_basebackup makes an exact copy of the database cluster's files, while making sure the server is put into and out of backup mode automatically.

所以这个问题几乎等价于:

**x86 架构下的 pg 数据是否可以直接用 arm 架构下的 `postgres` 进程来管理？**

群里的讨论中出现了两种对立的观点:

- 一种认为字节序、对齐(padding)会导致不同架构数据格式的不兼容
- 一种认为理论上数据库的文件格式可以独立于 cpu 架构，存储格式的字节序跟 cpu 字节序没有直接关系

从 Postgres 的实现来说，第一种观点是正确的；但如果抛开 PG，第二种观点我认为也是一定成立的。

**先说第二种观点:**

![](/images/2024/IMG_8559.jpg)

其实这种观点是有依据的，我举一个例子:

[Parquet](https://parquet.apache.org/) 是一种跨平台的数据存储格式，可以在不同的 CPU 架构上使用。这是因为 Parquet 的设计不依赖特定的硬件架构或操作系统，而是通过定义数据的存储结构和元数据来实现数据的高效读写和处理，并提供相关的库对数据进行序列化和反序列化。PG 生态中读写 Parquet 的 FDW 正是通过这些库来实现的。

只不过在 PG 的群里大家普遍会从 Postgres 的角度去考虑问题，因此这种观点不太容易被认同（but I get it 😜）。

**再看第一种观点:**

不同于 Parquet/ORC 等开放文件格式（open file format），Postgres 的文件格式是专有的（proprietary
），它的设计更专注于性能，使用 buffer cache 来存储最近或频繁访问的数据页，以减少对物理存储的访问次数。buffer cache 中的页与文件系统中的数据文件块直接映射，这样可以快速地在内存和磁盘之间交换数据。

> Data layout on disk fully coincides with data representation in RAM. The page along
with its tuples is read into the buffer cache **as is**, without any transformations.
That’s why data files are incompatible between different platforms.

不兼容的原因之一是字节序。x86 架构是小端序，IBM z/Architecture 是大端序，而 ARM 具有可配置的字节序。既然我们已经知道 PG 的数据在内存和磁盘之间不进行任何转换，自然意味着 **Postgres 的数据文件保留了其实例所在主机 CPU 的字节序**，x86 下的数据文件显然不能被小端序架构下的 postgres 进程直接读写。

另一个不兼容的原因是数据通常会对齐到机器字边界。例如，在 32 位的 x86 系统中，int 和 double 都会按照四字节字边界对齐，然而在 64 位系统中，double 会按照八字节对齐。数据对齐使得元组的大小依赖于表中字段的顺序。这也导致了不同位宽的机器的 PG 数据不兼容。

举个简单的例子:

```SQL
=> CREATE EXTENSION IF NOT EXISTS pageinspect;

=> CREATE TABLE padding1(
    c0 integer,
    c1 integer,
    c2 double precision
);
=> INSERT INTO padding1 VALUES (0,1,2.0);
=> SELECT lp_len FROM heap_page_items(get_raw_page('padding1', 0));
┌────────┐
│ lp_len │
├────────┤
│     40 │
└────────┘
(1 row)

=> CREATE TABLE padding2(
    c0 integer,
    c2 double precision,
    c1 integer
);
=> INSERT INTO padding2 VALUES (0,2.0,1);
=> SELECT lp_len FROM heap_page_items(get_raw_page('padding2', 0));
┌────────┐
│ lp_len │
├────────┤
│     44 │
└────────┘
(1 row)
```

**结论**

最好不要用 `pg_basebackup` 在不同的 CPU 架构之间去做数据同步，如果真的需要做异构平台的数据复制，应该使用 [Logical Replication](https://www.postgresql.org/docs/current/logical-replication.html) 或 [pg_dump](https://www.postgresql.org/docs/current/app-pgdump.html)，并在上生产前做好充分的验证。

**思考**

回到文章开头的问题，假设 arm 配置成跟 x86 一样的小端序，并且位宽也一样（padding 规则一致），那么这个问题的答案是否会有所不同？

### References

- [pg_basebackup](https://www.postgresql.org/docs/current/app-pgbasebackup.html)
- [PostgreSQL 14 internals | 3 Pages and Tuples](https://edu.postgrespro.com/postgresql_internals-14_en.pdf)
