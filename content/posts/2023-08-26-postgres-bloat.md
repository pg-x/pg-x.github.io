---
author: pg-x
title: "PostgreSQL 数据膨胀解决方法不完全指南"
date: 2023-08-26T10:00:36+08:00
draft: false
tags: ["bloat"]
ShowToc: false
TocOpen: false
---

PostgreSQL 中的数据膨胀（bloat）是指数据表或者索引中无用数据过多占用空间的情况。PostgreSQL 通过元组中的 xmin/xmax 以及可见性判断逻辑来实现 MVCC，当更新或删除一条元组的时候，会更新它的 xmax 来标记删除而非真正进行物理删除，因此如果大量死元组未及时清理，会出现数据膨胀，占用额外存储空间且影响查询性能。本文我们讨论数据膨胀的主要原因以及如何检查并解决的方法。

### 膨胀原因

PostgreSQL 中的 autovacuum launcher 在满足条件时会启动 autovacuum worker 去清理垃圾数据，但当出现以下情况时，依旧会出现膨胀的现象:

- 设置了 `autovauum_enabled = false`，防止 autovacuum 进程清理该表（但如果表 age 过大，autovacuum 依然会处理该表以防止 `transaction ID wraparound` 问题出现）
- 长事务，如果一个事务长时间未结束，会阻止 vacuum 进程清理该事务 id 之后修改或删除的元组
- 更新/删除频繁的业务负载
- autovacuum 参数配置不合理

### 检查是否存在膨胀

检查数据膨胀的方法有很多，常用于监控或异常时手动检查，下面列举几种检查的方法。

#### pg_stat_user_tables

`pg_stat_user_tables` 提供了用户表各种运行状态信息，其中 `n_live_tup` 和 `n_dead_tup` 表示一张表中活元组和死元组近似数值，以下查询可以检测数据膨胀情况:

```SQL
SELECT schemaname, relname, n_live_tup, n_dead_tup, 
       last_vacuum, last_autovacuum, vacuum_count, autovacuum_count
FROM pg_stat_user_tables
ORDER BY 2, 3 DESC;
```

上述查询还会输出一些跟 vacuum 相关的字段以检测 vacuum 在该表上运行的情况。`pg_stat_user_tables` 还可以查看 seq_scan 和 idx_scan 的统计信息以帮助用户更好地优化索引的创建。

#### pgstattuple

pgstattuple 是一个 PostgreSQL 自带的插件，它提供了一些用于获取有关表和索引的物理统计信息（包括空闲空间，碎片，以及元组大小等）的函数。常用的三个函数:

- pgstattuple(relname): 对表文件（包括 heap，btree, hash, gist）逐页访问，根据页面头信息和元组头信息构造统计信息
- pgstatindex(indexname): 返回指定 btree 索引的统计信息，包括树的高度、叶子节点个数、叶子数据密度等
- pgstattuple_approx(relname): 只对 heap 有效，逐页访问，对于页面元组全部可见的页，只统计元组长度，不统计个数，因此结果是预估值，但相对 pgstattuple 要快一些

下面用一个例子展示一下它们各自的输出:

```SQL
postgres=# create table tbl(id int) with (autovacuum_enabled = false);  -- 对该表关闭 autovacuum
CREATE TABLE
postgres=# create index tbl_idx on tbl(id);
CREATE INDEX
postgres=# insert into tbl select * from generate_series(1,100000);
INSERT 0 100000
postgres=# \x
Expanded display is on.
postgres=# SELECT pg_size_pretty(pg_relation_size('tbl')), pg_size_pretty(pg_relation_size('tbl_idx'));;
-[ RECORD 1 ]--+--------
pg_size_pretty | 3544 kB
pg_size_pretty | 2208 kB

postgres=# create extension pgstattuple ;
CREATE EXTENSION
postgres=# select * from pgstattuple('tbl');
-[ RECORD 1 ]------+--------
table_len          | 3629056
tuple_count        | 100000
tuple_len          | 2800000
tuple_percent      | 77.16
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 16652
free_percent       | 0.46

postgres=# select * from pgstattuple_approx('tbl');
-[ RECORD 1 ]--------+-------------------
table_len            | 3629056
scanned_percent      | 100
approx_tuple_count   | 100000
approx_tuple_len     | 2800000
approx_tuple_percent | 77.15505079006772
dead_tuple_count     | 0
dead_tuple_len       | 0
dead_tuple_percent   | 0
approx_free_space    | 16652
approx_free_percent  | 0.4588521091986456
```

可以看出估算值跟精确值很接近，下面是索引的一些统计信息，叶子节点的数据密度高达 89.83%，即无用数据占 10% 左右的空间，这是一个好现象（索引默认的 fillfactor 是 90%）。

```SQL
postgres=# select * from pgstattuple('tbl_idx');
-[ RECORD 1 ]------+--------
table_len          | 2260992
tuple_count        | 100000
tuple_len          | 1600000
tuple_percent      | 70.77
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 227092
free_percent       | 10.04

postgres=# select * from pgstatindex('tbl_idx');
-[ RECORD 1 ]------+--------
version            | 4
tree_level         | 1
index_size         | 2260992
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 274
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 89.83
leaf_fragmentation | 0
```

然后删除一部分数据再来看它们的输出:

```SQL
postgres=# delete from tbl where id % 2 = 0 or id < 9999;
DELETE 54999
postgres=# select * from pgstattuple('tbl');
-[ RECORD 1 ]------+--------
table_len          | 3629056
tuple_count        | 45001
tuple_len          | 1260028
tuple_percent      | 34.72
dead_tuple_count   | 54999
dead_tuple_len     | 1539972
dead_tuple_percent | 42.43
free_space         | 16652
free_percent       | 0.46

postgres=# select * from pgstattuple_approx('tbl');
-[ RECORD 1 ]--------+-------------------
table_len            | 3629056
scanned_percent      | 100
approx_tuple_count   | 45001
approx_tuple_len     | 1260028
approx_tuple_percent | 34.72054440603837
dead_tuple_count     | 54999
dead_tuple_len       | 1539972
dead_tuple_percent   | 42.43450638402935
approx_free_space    | 16652
approx_free_percent  | 0.4588521091986456
```

可以看到删除的元组已经被统计到相关字段，并且精确值和估计值的统计依旧很接近。再看索引，由于索引上没有可见性信息，删除操作并未修改索引的结构。

```SQL
postgres=# select * from pgstattuple('tbl_idx');
-[ RECORD 1 ]------+--------
table_len          | 2260992
tuple_count        | 100000
tuple_len          | 1600000
tuple_percent      | 70.77
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 227092
free_percent       | 10.04

postgres=# select * from pgstatindex('tbl_idx');
-[ RECORD 1 ]------+--------
version            | 4
tree_level         | 1
index_size         | 2260992
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 274
empty_pages        | 0
deleted_pages      | 0
avg_leaf_density   | 89.83
leaf_fragmentation | 0
```

然后我们对表进行 Vacuum:

```SQL
postgres=# vacuum tbl;
VACUUM
postgres=# select * from pgstattuple('tbl');
-[ RECORD 1 ]------+--------
table_len          | 3629056
tuple_count        | 45001
tuple_len          | 1260028
tuple_percent      | 34.72
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 1817816
free_percent       | 50.09

postgres=# select * from pgstattuple_approx('tbl');
-[ RECORD 1 ]--------+-------------------
table_len            | 3629056
scanned_percent      | 0
approx_tuple_count   | 45001
approx_tuple_len     | 1811264
approx_tuple_percent | 49.910059255079005
dead_tuple_count     | 0
dead_tuple_len       | 0
dead_tuple_percent   | 0
approx_free_space    | 1817792
approx_free_percent  | 50.089940744920995

postgres=# select * from pgstattuple('tbl_idx');
-[ RECORD 1 ]------+--------
table_len          | 2260992
tuple_count        | 45001
tuple_len          | 720016
tuple_percent      | 31.85
dead_tuple_count   | 0
dead_tuple_len     | 0
dead_tuple_percent | 0
free_space         | 1328800
free_percent       | 58.77

postgres=# select * from pgstatindex('tbl_idx');
-[ RECORD 1 ]------+--------
version            | 4
tree_level         | 1
index_size         | 2260992
root_block_no      | 3
internal_pages     | 1
leaf_pages         | 247
empty_pages        | 0
deleted_pages      | 27
avg_leaf_density   | 44.99
leaf_fragmentation | 0
```

可以看出，heap 表或索引对应的文件大小不变，但由于删除了死元组所占空间，页面出现了空洞，索引叶子节点的数据密度也降低了，这些对于扫面查询的性能会有一定的负面影响。

需要注意的是，pgstattuple 插件提供的函数都是对整张表进行了扫面，在数据量很大的情况下，可能会比较耗时。


### 数据膨胀合理值

那多少膨胀数据是一个可接受的值，不同的业务场景可能会有各自的答案。Citus Con 2023 Chelsea Dole 在她的 [Understanding & Managing Postgres Table Bloat](https://www.youtube.com/watch?v=gAgbzvGT6ck&t=1053s) 中给出一个经验值:


| Table Size  | rule of thumb |
| ----- | --- |
| Very Small (<=1GB) | Up to 70% bloat is acceptable |
| Small (1-30GB) | Up to 25% dead tuples is acceptable |
| Large (30-100GB) | Up to 20% dead tuples is acceptable |
| Very Large (>100GB) | Up to 18% dead tuples is acceptable |

### 解决方案

PostgreSQL 通过一些机制减少数据膨胀的几率，如 [HOT 机制](https://www.postgresql.org/docs/15/storage-hot.html) 可以减少索引中垃圾数据比例；PG14 引入 [Bottom-up Index Deletion](https://www.postgresql.org/docs/14/btree-implementation.html)，在索引页分裂前清除指向死元组的索引项来避免昂贵的分裂操作，参见 [INDEX BLOAT REDUCED IN POSTGRESQL V14](https://www.cybertec-postgresql.com/en/index-bloat-reduced-in-postgresql-v14/) 和 [Bloat in PostgreSQL: a taxonomy](https://www.youtube.com/watch?v=JDG4bMHxCH8) 以了解更多。但即使有这些优化，生产环境依然存在数据膨胀的时候，下面针对造成 bloat 的不同原因讨论对应的解决方法。

#### autovacuum 参数配置不合理

在数据更新/删除频繁的工作负载下，可以通过设置更激进的参数来增加 autovauum 调度的频率。

- autovacuum_vacuum_scale_factor
  - 在表级别将默认值 0.2 调整为 0.01，这样当表的 1% 被修改就会触发 vacuum
- autovacuum_max_workers
  - 如果有大量的表，则可以查看 pg_stat_progress_vacuum 有多少并行的 vacuum worker 并适当增大 autovacuum_max_workers 值


#### 长事务阻塞 autovacuum

用下面的语句查询当前是否有 `idle in transaction` 的事务或 **long-running query**，找到相关的应用进行处理。

```SQL
SELECT pid, state, query, xact_start, now() - xact_start AS duration
FROM pg_stat_activity
WHERE state like '%transaction%' or state = 'active'
ORDER by 5 DESC LIMIT 5;
```

#### 消除数据空洞

由于 Vacuum 只整理页面内部的垃圾数据，会出现少量元组分布在大量数据页中的情况，在需要 seq scan 的时候需要将磁盘页面 load 到内存中，会有大量的 IO 操作，这依然不是一个好现象，此时需要用 `VACUUM FULL` 来重写一份紧凑的数据。

#### VACUUM FULL 失败

由于 `VACUUM FULL` 需要对整张表加写锁，在此过程写操作不能进行，这在有的业务场景中是不能允许的，此时可以考虑 [pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze) 插件，它借助 [Logical Decoding](https://www.postgresql.org/docs/15/logicaldecoding.html) 的能力可以在重写表文件的时候不对原始表加锁，本文不过多描述，如感兴趣请自行查看。

### 小结

PostgreSQL 中的数据膨胀在 heap 表和索引中都会存在，合理的 autovacuum 参数能减少膨胀对性能的影响，社区也通过一些优化减少数据膨胀的比例，当遇到不正常的数据膨胀时，可通过本文介绍的方法查看问题原因，另外autovacuum 的日志也是查找问题的一个重要线索。
