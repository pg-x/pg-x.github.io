---
author: pg-x
title: "PostgreSQL 17 增量备份"
date: 2024-01-20T07:52:59+08:00
tags: ["Waiting for PG17"]
ShowToc: false
TocOpen: false
---

PostgreSQL 17 的一个重要特性是在内核层面增加了增量备份（Incremental Backup）的特性，为用户提供更灵活和高效的备份解决方案。

该特性由 EDB 的首席数据库架构师 [Robert Haas](https://www.linkedin.com/in/robertmhaas/) 实现，详细讨论见: [trying again to get incremental backup](https://www.postgresql.org/message-id/flat/CA%2BTgmoYOYZfMCyOXFyC-P%2B-mdrZqm5pP2N7S-r0z3_402h9rsA%40mail.gmail.com)

### Why Incremental Backup?

PostgreSQL 通过全量备份和 WAL archiving 能够实现 PITR(Point-In-Time Recovery)，从功能完备性角度来讲，也许并不需要增量备份。但对一个 10T 数据量、读多写少的实例，每周（月）进行一次全量备份可能太过繁重。

增量备份则仅备份自上次备份（可以是全量备份或增量备份）以来修改过的文件页面（8K Block），对于数据量大且修改频率较低的数据库实例，增量备份有以下优势：

- 快速备份：由于增量备份只包含自上次备份以来的变化部分，因此备份速度更快。相比全量备份，增量备份需要拷贝的数据量更小，能够更快地完成备份过程。
- 空间效率：由于增量备份仅存储自上次备份以来的更改部分，所需的磁盘空间较少，节省存储成本。
- 快速恢复：在数据丢失或灾难恢复的情况下，从**全量+增量备份**进行恢复通常比从**全量备份+WAL**进行恢复更快。将最近的全量备份和后续的增量备份进行合并，相比回放大量的 WAL 日志，恢复时间更短，可以快速使系统恢复正常运行。

EDB 工程师 Jakub Wartak 进行的测试证明了增量备份的必要性:

![incremental backup test](/images/2024/incremental_backup_test.png)

对一个初始数据量 3.5G 的实例用 `pg_bench` 持续写入，24 小时生成的 WAL 数据量 77GB，最后一次的增量备份数据量 3.5GB，相对最终的总数据量 4.3GB 差距不明显，说明大部分页面有过改动。

**全量+增量备份** 恢复的速度（4min18s）却远快于**全量备份+WAL**（78min）。

当然，增量备份也并非银弹，[增量备份文档](https://www.postgresql.org/docs/devel/continuous-archiving.html#BACKUP-INCREMENTAL-BACKUP)中有这样一段描述:

> Incremental backups typically only make sense for **relatively large databases where a significant portion of the data does not change, or only changes slowly**. For a small database, it's simpler to ignore the existence of incremental backups and simply take full backups, which are simpler to manage. For a large database all of which is heavily modified, incremental backups won't be much smaller than full backups.

### How?

PostgreSQL 是一种典型的段页式（Segment/Block）关系型数据库管理系统，增量备份的实现依赖能够准确识别自上次备份以来哪些页面发生了修改，Robert Haas 在 PGCon 2022 [Moving pg_basebackup Forward](https://www.youtube.com/watch?v=PXzC18cle4Q) 中提到了五种识别页面变更的方式，每种方式都有各自的问题，Robert 在实现上最终采用了第四种方式。

![Incremental backup ](/images/2024/IMG_7700.jpg)

整体设计上分为三部分:

- 首先，引入一个名为 `walsummarizer` 的后端进程。它读取 WAL（Write-Ahead Log）并生成 WAL summary 文件。相比原始 WAL 文件，WAL summary 文件非常小，仅包含特定区间内数据库修改的摘要信息，如有关文件的创建、删除或截断以及修改的页面信息等。
- 其次，`pg_basebackup` 增加了增量备份模式。通过解析前一次备份的 backup_manifest 获取上一个备份开始和本次备份开始之间生成的 WAL summary 文件，并用它们来识别哪些关系文件的页面（relno + fork + block）发生过更改。
- 第三，一个名为 `pg_combinebackup` 的程序，将一个全量备份和一个或多个增量备份，通过执行一系列的完整性检查后进行合并，如果一切正常，会生成一个新的全量备份。

下面逐一介绍。

### 1. Wal Summarizer

通过读取一段 LSN 范围的 WAL 日志，我们能够知道哪些页面发生了变化（某种意义上的 logical decoding），但如果每次增量备份都去读取 WAL 日志，存在两个问题:

- 读取和解析 WAL 日志需要的时间相对较长
- 需要保留更多的 WAL 日志

因此增加一个后端进程持续收集页面变更记录是一个巧妙的设计。`walsummarizer` 进程由 *summarize_wal* 参数控制是否启动，默认不开启，wal_level = minimal 时 `walsummarizer` 进程也不启动。

将该参数打开:

```SQL
postgres=# show summarize_wal;
 summarize_wal
---------------
 off
(1 row)

postgres=# ALTER SYSTEM SET summarize_wal = on;
ALTER SYSTEM
postgres=# SELECT pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=# show summarize_wal;
 summarize_wal
---------------
 on
(1 row)
```

walsummarizer 启动:

![walsummarizer](/images/2024/walsummarizer_be.png)

该进程读取 WAL 日志并生成 summary 文件:

```shell
➜  postgres_data ls -al pg_wal/summaries
total 40
drwx------@ 7 zhjwpku  staff   224 Jan 18 21:22 .
drwx------@ 6 zhjwpku  staff   192 Jan 18 21:24 ..
-rw-------@ 1 zhjwpku  staff    32 Jan 18 21:20 0000000100000000030000280000000003000100.summary
-rw-------@ 1 zhjwpku  staff    32 Jan 18 21:20 0000000100000000030001000000000003000208.summary
-rw-------@ 1 zhjwpku  staff    32 Jan 18 21:20 00000001000000000300020800000000030002B8.summary
-rw-------@ 1 zhjwpku  staff   998 Jan 18 21:20 0000000100000000030002B800000000031BEBA8.summary
-rw-------@ 1 zhjwpku  staff  3278 Jan 18 21:22 0000000100000000031BEBA80000000004053600.summary
```

生成的规则是当读取到 `XLOG_CHECKPOINT_REDO` 或 `XLOG_CHECKPOINT_SHUTDOWN` 记录后生成一个新的文件，格式为 `$TLI${START_LSN}${END_LSN}.summary`。文件名也用于后续解析该文件记录的 LSN 范围:

```C
sscanf(dent->d_name, "%08X%08X%08X%08X%08X",
        &tmp[0], &tmp[1], &tmp[2], &tmp[3], &tmp[4]);
file_tli = tmp[0];
file_start_lsn = ((uint64) tmp[1]) << 32 | tmp[2];
file_end_lsn = ((uint64) tmp[3]) << 32 | tmp[4];
```

用 `pg_available_wal_summaries` UDF 来查看效果更好:

```SQL
postgres=# select pg_available_wal_summaries();
 pg_available_wal_summaries
----------------------------
 (1,0/3000100,0/3000208)
 (1,0/30002B8,0/31BEBA8)
 (1,0/3000208,0/30002B8)
 (1,0/3000028,0/3000100)
 (1,0/31BEBA8,0/4053600)
(5 rows)
```

如果两个 Checkpoint 之间没有任何修改，对应的 summary 文件则不记录任何变更。如: 0000000100000000030000280000000003000100.summary。可以用 `pg_walsummary` 工具或 `pg_wal_summary_contents` UDF 来查看 summary 中记录的内容:

```
➜  postgres_data pg_walsummary pg_wal/summaries/0000000100000000030000280000000003000100.summary
➜  postgres_data psql postgres
psql (17devel)
Type "help" for help.
postgres=# select pg_wal_summary_contents(1,'0/3000028','0/3000100');
 pg_wal_summary_contents
-------------------------
(0 rows)
```

summary 文件是一种非常节省空间（space-efficient）的二进制格式，且在磁盘和内存之间传输非常方便，对其格式的介绍超出了本文讨论的范围，感兴趣的同学请阅读 **src/common/blkreftable.c** 文件。

下面我们对数据库进行少量变更，观察新生成的 summary 文件中记录了什么内容。

```SQL
postgres=# \d+ t1
                                           Table "public.t1"
 Column |  Type  | Collation | Nullable | Default | Storage  | Compression | Stats target | Description
--------+--------+-----------+----------+---------+----------+-------------+--------------+-------------
 id     | bigint |           |          |         | plain    |             |              |
 name   | text   |           |          |         | extended |             |              |
Access method: heap

postgres=# \d+
                                  List of relations
 Schema | Name | Type  |  Owner  | Persistence | Access method | Size  | Description
--------+------+-------+---------+-------------+---------------+-------+-------------
 public | t1   | table | zhjwpku | permanent   | heap          | 14 MB |
(1 row)

postgres=# select min(id), max(id) from t1;
 min |  max
-----+--------
   1 | 100000
(1 row)

postgres=# update t1 set name = repeat('b', 100) where id % 1000 = 0;
UPDATE 100
postgres=# checkpoint;
CHECKPOINT
```

查看新生成的 summary 文件内容:

```SQL
postgres=# select pg_available_wal_summaries();
 pg_available_wal_summaries
----------------------------
 (1,0/3000100,0/3000208)
 (1,0/30002B8,0/31BEBA8)
 (1,0/4053600,0/4120FB8)        <--- 新增的 summary
 (1,0/3000208,0/30002B8)
 (1,0/3000028,0/3000100)
 (1,0/31BEBA8,0/4053600)
(6 rows)

postgres=# select pg_wal_summary_contents(1,'0/4053600','0/4120FB8') limit 10;
 pg_wal_summary_contents
-------------------------
 (2619,1663,5,0,16,f)       <--- (relNumber, spcOid, dbOid, forknum, limit_block/block_num, is_limit_block)
 (16384,1663,5,0,17,f)
 (16384,1663,5,0,1724,f)
 (16384,1663,5,0,34,f)
 (16384,1663,5,0,51,f)
 (16384,1663,5,0,68,f)
 (16384,1663,5,0,86,f)
 (16384,1663,5,0,103,f)
 (16384,1663,5,0,120,f)
 (16384,1663,5,0,137,f)
(10 rows)

postgres=# select count(1) from pg_wal_summary_contents(1,'0/4053600','0/4120FB8') limit 10;
 count
-------
   102
(1 row)
```

上面的 update 语句修改了 102 个页面，增量备份正是基于这些内容来找到需要拷贝的页面。

当然，summary 也有留存期限，通过 `wal_summary_keep_time` 控制，默认 10 天，当增量备份所需的 summary 文件被删除了，则需要重新做一次全量备份。

summary 相关的 commit: [Add a new WAL summarizer process](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=174c480508ac25568561443e6d4a82d5c1103487)

### 2. Incremental backup

增量备份是在 pg_basebackup 工具中新增了 `--incremental` 选项，选项后跟上次备份生成的 backup_manifest 文件，该备份即可以是全量备份，也可以是另一个增量备份。为了方便演示，我们先生成一个全量备份:

```
➜  fake_root pg_basebackup -cfast -Dx
➜  fake_root ls x/pg_wal/summaries     <--- summary 文件未备份
➜  fake_root cat x/backup_manifest     <--- backup_manifest 可以用来验证备份的完整性
{ "PostgreSQL-Backup-Manifest-Version": 1,
"Files": [
{ "Path": "backup_label", "Size": 225, "Last-Modified": "2024-01-18 15:12:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "7990104a" },
{ "Path": "pg_multixact/members/0000", "Size": 8192, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
{ "Path": "pg_multixact/offsets/0000", "Size": 8192, "Last-Modified": "2024-01-18 13:11:52 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
{ "Path": "PG_VERSION", "Size": 3, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "64440205" },
{ "Path": "pg_hba.conf", "Size": 5711, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "d62da38c" },
{ "Path": "pg_logical/replorigin_checkpoint", "Size": 8, "Last-Modified": "2024-01-18 15:12:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "c74b6748" },
{ "Path": "postgresql.conf", "Size": 29795, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "d7ffcf81" },
{ "Path": "postgresql.auto.conf", "Size": 109, "Last-Modified": "2024-01-18 13:20:19 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "1259dae3" },
{ "Path": "logfile", "Size": 6860, "Last-Modified": "2024-01-18 15:12:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "9b3b822e" },
{ "Path": "pg_xact/0000", "Size": 8192, "Last-Modified": "2024-01-18 14:20:49 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "64fc020b" },
{ "Path": "pg_ident.conf", "Size": 2640, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "0ce04d87" },
{ "Path": "global/4178", "Size": 8192, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "3c59e3c5" },
... skip skip
{ "Path": "base/5/2664", "Size": 16384, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "f673abad" },
{ "Path": "base/5/1249_vm", "Size": 8192, "Last-Modified": "2024-01-18 13:16:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "cf64d329" },
{ "Path": "base/5/2836", "Size": 8192, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "d65a10e9" },
{ "Path": "base/5/2663", "Size": 40960, "Last-Modified": "2024-01-18 13:16:54 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "57e215d9" },
{ "Path": "base/5/6237", "Size": 0, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "00000000" },
{ "Path": "global/pg_control", "Size": 8192, "Last-Modified": "2024-01-18 15:12:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "43872087" }
],
"WAL-Ranges": [
{ "Timeline": 1, "Start-LSN": "0/7000028", "End-LSN": "0/7000120" }
],
"Manifest-Checksum": "6074f5951576bf544623864712639d80e87c5dd0e557a367c6ec98028822fca3"}
➜  fake_root
```

对于增量备份，backup_manifest 中最重要的信息是 "WAL-Ranges"。pg_basebackup 为了将 backup_manifest 传给后端，在复制协议中增加了 `UPLOAD_MANIFEST` 命令，整个文件传输完毕后对 JSON 进行解析，如果 summaries 目录中的记录能覆盖上次备份开始到本次备份开始的 LSN 区间，则读取对应的 summary 文件，根据记录信息将一些关系文件替换为 **INCREMENTAL.${ORIGINAL_NAME}** 文件并拷贝相应的变更页面到该文件中。

> We need WAL summaries for everything that happened during the prior
> backup and everything that happened afterward up until the point where
> the current backup started.

然后基于 x 生成增量备份 y:

```
➜  fake_root pg_basebackup -cfast -Dy --incremental x/backup_manifest
➜  fake_root cat y/backup_manifest
{ "PostgreSQL-Backup-Manifest-Version": 1,
"Files": [
{ "Path": "backup_label", "Size": 281, "Last-Modified": "2024-01-18 15:26:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "10b486a5" },
{ "Path": "pg_multixact/members/0000", "Size": 8192, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
{ "Path": "pg_multixact/offsets/0000", "Size": 8192, "Last-Modified": "2024-01-18 13:11:52 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
{ "Path": "PG_VERSION", "Size": 3, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "64440205" },
{ "Path": "pg_hba.conf", "Size": 5711, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "d62da38c" },
{ "Path": "pg_logical/replorigin_checkpoint", "Size": 8, "Last-Modified": "2024-01-18 15:26:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "c74b6748" },
{ "Path": "postgresql.conf", "Size": 29795, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "d7ffcf81" },
{ "Path": "postgresql.auto.conf", "Size": 109, "Last-Modified": "2024-01-18 13:20:19 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "1259dae3" },
{ "Path": "logfile", "Size": 7981, "Last-Modified": "2024-01-18 15:26:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "3975dac0" },
{ "Path": "pg_xact/0000", "Size": 8192, "Last-Modified": "2024-01-18 15:25:57 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "656f942a" },
{ "Path": "pg_ident.conf", "Size": 2640, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "0ce04d87" },
{ "Path": "global/INCREMENTAL.4178", "Size": 12, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "global/4185", "Size": 0, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "00000000" },
{ "Path": "global/INCREMENTAL.6245", "Size": 12, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "global/INCREMENTAL.4176", "Size": 12, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "global/INCREMENTAL.4182", "Size": 12, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "global/INCREMENTAL.3593", "Size": 12, "Last-Modified": "2024-01-16 02:50:51 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
... skip skip
{ "Path": "base/5/INCREMENTAL.2838", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "3d9af9b5" },
{ "Path": "base/5/INCREMENTAL.2652", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "e34c7d7c" },
{ "Path": "base/5/2840_fsm", "Size": 24576, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "e90d0c12" },
{ "Path": "base/5/INCREMENTAL.2831", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "base/5/INCREMENTAL.2690", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "773d7c4e" },
{ "Path": "base/5/INCREMENTAL.2664", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "e34c7d7c" },
{ "Path": "base/5/INCREMENTAL.1249_vm", "Size": 12, "Last-Modified": "2024-01-18 13:16:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "base/5/INCREMENTAL.2836", "Size": 12, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "dac55f1e" },
{ "Path": "base/5/INCREMENTAL.2663", "Size": 12, "Last-Modified": "2024-01-18 13:16:54 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "29f47d65" },
{ "Path": "base/5/6237", "Size": 0, "Last-Modified": "2024-01-16 02:50:53 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "00000000" },
{ "Path": "global/pg_control", "Size": 8192, "Last-Modified": "2024-01-18 15:26:41 GMT", "Checksum-Algorithm": "CRC32C", "Checksum": "43872087" }
],
"WAL-Ranges": [
{ "Timeline": 1, "Start-LSN": "0/9000028", "End-LSN": "0/9000120" }
],
"Manifest-Checksum": "d5f1cdf542d95226780b9df2eb34b3a31e8d2c53369a57f95a3ece6fe5bd27e9"}
➜  fake_root
```

从 backup_manifest 文件能够看出，增量备份除了一些关系文件被替换为 **INCREMENTAL.${ORIGINAL_NAME}** 文件外，其它全量备份的文件也都进行了拷贝，并且全量备份所有的约束也都适用于增量备份，比如仍需要拷贝在备份期间和之后生成的所有 WAL 文件，以及任何相关的 WAL 历史文件。另外，FSM 文件不做增量备份，而是直接拷贝。

在 PG 数据库实例中，由于关系文件占的比重最大，因而增量备份通常能减少存储空间:

```shell
➜  fake_root du -sh x      <--- x 为全量备份
 51M	x
➜  fake_root du -sh y      <--- y 为增量备份
 22M	y
```

由于 PostgreSQL 的 JSON parser 不支持增量解析，因而需要将整个 backup_manifest 传输到后端后才能开始解析，这就对增量备份的使用增加了一个额外的限制：**当 backup_manifest 文件大于 1GB 时，增量备份会失败。**

一个简单的估算，backup_manifest 文件中的每条记录大概 150 字节，1GB 的空间可以容纳 **1024^3/150 = 7158278** 个文件。将数据库使用到这个程度也非易事，所以 1GB 的限制看起来还好。

JSON parser 支持增量解析的特性由 Andrew Dunstan 开发，不过现在的状态是性能不够好，感兴趣的同学可以关注对应的邮件列表讨论。

值得一提的是，增量备份可以在 hot standby 上生成，这点跟全量备份保持一致。

增量备份相关的 commit: [Add support for incremental backup](https://git.postgresql.org/gitweb/?p=postgresql.git;a=commitdiff;h=dc212340058b4e7ecfc5a7a81ec50e7a207bf288)

### 3. pg_combinebackup

`pg_combinebackup` 工具用于将一个全量备份和一系列增量备份重建一个全量备份。比如，有一个完整备份 x、一个增量备份 y(基于 x) 和一个增量备份 z(基于 y)，你可以将 x、y 和 z 组合在一起，得到一个等效于在进行 z 备份时的全量备份。但是，pg_combinebackup 不能将 y 和 z 直接合并。

> The new pg_combinebackup tool can be used to reconstruct a data directory
from a full backup and a series of incremental backups.

使用 `pg_combinebackup` 时对参数顺序有一定要求，需按照**从最旧到最新**依次指定所需的备份，也就是说，第一个备份目录应该是全量备份的路径，而最后一个备份目录应该是希望恢复的最后一个增量备份的路径。

新生成的备份是一个全新的目录，由 **-o** 参数指定，`pg_combinebackup` 不支持原地合并。另外 **-d/--debug** 参数能够输出一系列调试信息来方便查询问题。

```
➜  fake_root pg_combinebackup -d x y -o z
pg_combinebackup: read server version 17 from "y/PG_VERSION"
pg_combinebackup: reading "y/global/pg_control"
pg_combinebackup: reading "x/global/pg_control"
pg_combinebackup: system identifier is 7324523201679734756
pg_combinebackup: reading "y/backup_label"
pg_combinebackup: reading "x/backup_label"
pg_combinebackup: scanning "y/pg_tblspc"
pg_combinebackup: creating directory "z"
pg_combinebackup: generating "z/backup_label"
pg_combinebackup: processing backup directory "y"
pg_combinebackup: creating directory "z/pg_multixact"
pg_combinebackup: creating directory "z/pg_multixact/members"
pg_combinebackup: copying "y/pg_multixact/members/0000" to "z/pg_multixact/members/0000"
pg_combinebackup: creating directory "z/pg_multixact/offsets"
pg_combinebackup: copying "y/pg_multixact/offsets/0000" to "z/pg_multixact/offsets/0000"
... skip skip
pg_combinebackup: copying "y/base/5/2608_fsm" to "z/base/5/2608_fsm"
pg_combinebackup: reconstructing "z/base/5/16384" (1727 blocks, checksum CRC32C)
pg_combinebackup: reconstruction plan: 0-16:x/base/5/16384@131072 17:y/base/5/INCREMENTAL.16384@420 18-33:x/base/5/16384@270336 34:y/base/5/INCREMENTAL.16384@8612 35-50:x/base/5/16384@409600 51:y/base/5/INCREMENTAL.16384@16804 52-67:x/base/5/16384@548864 68:y/base/5/INCREMENTAL.16384@24996 69-85:x/base/5/16384@696320 86:y/base/5/INCREMENTAL.16384@33188 87-102:x/base/5/16384@835584 103:y/base/5/INCREMENTAL.16384@41380 104-119:x/base/5/16384@974848 120:y/base/5/INCREMENTAL.16384@49572 121-136:x/base/5/16384@1114112 137:y/base/5/INCREMENTAL.16384@57764 138-154:x/base/5/16384@1261568 155:y/base/5/INCREMENTAL.16384@65956 156-171:x/base/5/16384@1400832 172:y/base/5/INCREMENTAL.16384@74148 173-188:x/base/5/16384@1540096 189:y/base/5/INCREMENTAL.16384@82340 190-205:x/base/5/16384@1679360 206:y/base/5/INCREMENTAL.16384@90532 207-223:x/base/5/16384@1826816 224:y/base/5/INCREMENTAL.16384@98724 225-240:x/base/5/16384@1966080 241:y/base/5/INCREMENTAL.16384@106916 242-257:x/base/5/16384@2105344 258:y/base/5/INCREMENTAL.16384@115108 259-274:x/base/5/16384@2244608
pg_combinebackup: reconstruction plan: 275:y/base/5/INCREMENTAL.16384@123300 276-292:x/base/5/16384@2392064 293:y/base/5/INCREMENTAL.16384@131492 294-309:x/base/5/16384@2531328 310:y/base/5/INCREMENTAL.16384@139684 311-326:x/base/5/16384@2670592 327:y/base/5/INCREMENTAL.16384@147876 328-343:x/base/5/16384@2809856 344:y/base/5/INCREMENTAL.16384@156068 345-361:x/base/5/16384@2957312 362:y/base/5/INCREMENTAL.16384@164260 363-378:x/base/5/16384@3096576 379:y/base/5/INCREMENTAL.16384@172452 380-395:x/base/5/16384@3235840 396:y/base/5/INCREMENTAL.16384@180644 397-412:x/base/5/16384@3375104 413:y/base/5/INCREMENTAL.16384@188836 414-430:x/base/5/16384@3522560 431:y/base/5/INCREMENTAL.16384@197028 432-447:x/base/5/16384@3661824 448:y/base/5/INCREMENTAL.16384@205220 449-464:x/base/5/16384@3801088 465:y/base/5/INCREMENTAL.16384@213412 466-481:x/base/5/16384@3940352 482:y/base/5/INCREMENTAL.16384@221604 483-498:x/base/5/16384@4079616 499:y/base/5/INCREMENTAL.16384@229796 500-516:x/base/5/16384@4227072 517:y/base/5/INCREMENTAL.16384@237988 518-533:x/base/5/16384@4366336
pg_combinebackup: reconstruction plan: 534:y/base/5/INCREMENTAL.16384@246180 535-550:x/base/5/16384@4505600 551:y/base/5/INCREMENTAL.16384@254372 552-567:x/base/5/16384@4644864 568:y/base/5/INCREMENTAL.16384@262564 569-585:x/base/5/16384@4792320 586:y/base/5/INCREMENTAL.16384@270756 587-602:x/base/5/16384@4931584 603:y/base/5/INCREMENTAL.16384@278948 604-619:x/base/5/16384@5070848 620:y/base/5/INCREMENTAL.16384@287140 621-636:x/base/5/16384@5210112 637:y/base/5/INCREMENTAL.16384@295332 638-654:x/base/5/16384@5357568 655:y/base/5/INCREMENTAL.16384@303524 656-671:x/base/5/16384@5496832 672:y/base/5/INCREMENTAL.16384@311716 673-688:x/base/5/16384@5636096 689:y/base/5/INCREMENTAL.16384@319908 690-705:x/base/5/16384@5775360 706:y/base/5/INCREMENTAL.16384@328100 707-723:x/base/5/16384@5922816 724:y/base/5/INCREMENTAL.16384@336292 725-740:x/base/5/16384@6062080 741:y/base/5/INCREMENTAL.16384@344484 742-757:x/base/5/16384@6201344 758:y/base/5/INCREMENTAL.16384@352676 759-774:x/base/5/16384@6340608 775:y/base/5/INCREMENTAL.16384@360868 776-792:x/base/5/16384@6488064
pg_combinebackup: reconstruction plan: 793:y/base/5/INCREMENTAL.16384@369060 794-809:x/base/5/16384@6627328 810:y/base/5/INCREMENTAL.16384@377252 811-826:x/base/5/16384@6766592 827:y/base/5/INCREMENTAL.16384@385444 828-843:x/base/5/16384@6905856 844:y/base/5/INCREMENTAL.16384@393636 845-861:x/base/5/16384@7053312 862:y/base/5/INCREMENTAL.16384@401828 863-878:x/base/5/16384@7192576 879:y/base/5/INCREMENTAL.16384@410020 880-895:x/base/5/16384@7331840 896:y/base/5/INCREMENTAL.16384@418212 897-912:x/base/5/16384@7471104 913:y/base/5/INCREMENTAL.16384@426404 914-930:x/base/5/16384@7618560 931:y/base/5/INCREMENTAL.16384@434596 932-947:x/base/5/16384@7757824 948:y/base/5/INCREMENTAL.16384@442788 949-964:x/base/5/16384@7897088 965:y/base/5/INCREMENTAL.16384@450980 966-981:x/base/5/16384@8036352 982:y/base/5/INCREMENTAL.16384@459172 983-998:x/base/5/16384@8175616 999:y/base/5/INCREMENTAL.16384@467364 1000-1016:x/base/5/16384@8323072 1017:y/base/5/INCREMENTAL.16384@475556 1018-1033:x/base/5/16384@8462336 1034:y/base/5/INCREMENTAL.16384@483748 1035-1050:x/base/5/16384@8601600
pg_combinebackup: reconstruction plan: 1051:y/base/5/INCREMENTAL.16384@491940 1052-1067:x/base/5/16384@8740864 1068:y/base/5/INCREMENTAL.16384@500132 1069-1085:x/base/5/16384@8888320 1086:y/base/5/INCREMENTAL.16384@508324 1087-1102:x/base/5/16384@9027584 1103:y/base/5/INCREMENTAL.16384@516516 1104-1119:x/base/5/16384@9166848 1120:y/base/5/INCREMENTAL.16384@524708 1121-1136:x/base/5/16384@9306112 1137:y/base/5/INCREMENTAL.16384@532900 1138-1154:x/base/5/16384@9453568 1155:y/base/5/INCREMENTAL.16384@541092 1156-1171:x/base/5/16384@9592832 1172:y/base/5/INCREMENTAL.16384@549284 1173-1188:x/base/5/16384@9732096 1189:y/base/5/INCREMENTAL.16384@557476 1190-1205:x/base/5/16384@9871360 1206:y/base/5/INCREMENTAL.16384@565668 1207-1223:x/base/5/16384@10018816 1224:y/base/5/INCREMENTAL.16384@573860 1225-1240:x/base/5/16384@10158080 1241:y/base/5/INCREMENTAL.16384@582052 1242-1257:x/base/5/16384@10297344 1258:y/base/5/INCREMENTAL.16384@590244 1259-1274:x/base/5/16384@10436608 1275:y/base/5/INCREMENTAL.16384@598436 1276-1292:x/base/5/16384@10584064 1293:y/base/5/INCREMENTAL.16384@606628
pg_combinebackup: reconstruction plan: 1294-1309:x/base/5/16384@10723328 1310:y/base/5/INCREMENTAL.16384@614820 1311-1326:x/base/5/16384@10862592 1327:y/base/5/INCREMENTAL.16384@623012 1328-1343:x/base/5/16384@11001856 1344:y/base/5/INCREMENTAL.16384@631204 1345-1361:x/base/5/16384@11149312 1362:y/base/5/INCREMENTAL.16384@639396 1363-1378:x/base/5/16384@11288576 1379:y/base/5/INCREMENTAL.16384@647588 1380-1395:x/base/5/16384@11427840 1396:y/base/5/INCREMENTAL.16384@655780 1397-1412:x/base/5/16384@11567104 1413:y/base/5/INCREMENTAL.16384@663972 1414-1430:x/base/5/16384@11714560 1431:y/base/5/INCREMENTAL.16384@672164 1432-1447:x/base/5/16384@11853824 1448:y/base/5/INCREMENTAL.16384@680356 1449-1464:x/base/5/16384@11993088 1465:y/base/5/INCREMENTAL.16384@688548 1466-1481:x/base/5/16384@12132352 1482:y/base/5/INCREMENTAL.16384@696740 1483-1498:x/base/5/16384@12271616 1499:y/base/5/INCREMENTAL.16384@704932 1500-1516:x/base/5/16384@12419072 1517:y/base/5/INCREMENTAL.16384@713124 1518-1533:x/base/5/16384@12558336 1534:y/base/5/INCREMENTAL.16384@721316 1535-1550:x/base/5/16384@12697600
pg_combinebackup: reconstruction plan: 1551:y/base/5/INCREMENTAL.16384@729508 1552-1567:x/base/5/16384@12836864 1568:y/base/5/INCREMENTAL.16384@737700 1569-1585:x/base/5/16384@12984320 1586:y/base/5/INCREMENTAL.16384@745892 1587-1602:x/base/5/16384@13123584 1603:y/base/5/INCREMENTAL.16384@754084 1604-1619:x/base/5/16384@13262848 1620:y/base/5/INCREMENTAL.16384@762276 1621-1636:x/base/5/16384@13402112 1637:y/base/5/INCREMENTAL.16384@770468 1638-1654:x/base/5/16384@13549568 1655:y/base/5/INCREMENTAL.16384@778660 1656-1671:x/base/5/16384@13688832 1672:y/base/5/INCREMENTAL.16384@786852 1673-1688:x/base/5/16384@13828096 1689:y/base/5/INCREMENTAL.16384@795044 1690-1705:x/base/5/16384@13967360 1706:y/base/5/INCREMENTAL.16384@803236 1707-1723:x/base/5/16384@14114816 1724-1726:y/base/5/INCREMENTAL.16384@827812
pg_combinebackup: read 1625 blocks from "x/base/5/16384"
pg_combinebackup: read 102 blocks from "y/base/5/INCREMENTAL.16384"
... skip skip
pg_combinebackup: copying "x/base/5/5002" to "z/base/5/5002"
pg_combinebackup: recursively fsyncing "z"
➜  fake_root cat z/backup_manifest     <--- 查看合并后的 backup_manifest
{ "PostgreSQL-Backup-Manifest-Version": 1,
"Files": [
{ "Path": "backup_label", "Size": 225, "Last-Modified": "2024-01-19 01:56:41 UTC", "Checksum-Algorithm": "CRC32C", "Checksum": "8874f1d7" },
{ "Path": "pg_multixact/members/0000", "Size": 8192, "Last-Modified": "2024-01-19 01:56:41 UTC", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
{ "Path": "pg_multixact/offsets/0000", "Size": 8192, "Last-Modified": "2024-01-19 01:56:41 UTC", "Checksum-Algorithm": "CRC32C", "Checksum": "23464490" },
... skip skip
],
"WAL-Ranges": [
{ "Timeline": 1, "Start-LSN": "0/9000028", "End-LSN": "0/9000120" }
],
"Manifest-Checksum": "556eb55caf8f8b68a02beac95630542f001c2f21f5141b96fcfc2afe36db5742"}
➜  fake_root
```

pg_combinebackup 生成的 backup_manifest 中记录的 WAL-Ranges 信息和最后一个增量备份的 WAL-Ranges 相同。pg_combinebackup 的输出是一个等效的全量备份，可以基于它再去做增量备份。

合并的过程是从命令行指定的最后一个增量备份目录开始，递归地遍历目录并重构每一个文件，如果文件名不带 INCREMENTAL 前缀，直接拷贝到目标目录，对于带 INCREMENTAL 前缀的文件，读取最新的增量文件的头信息以及它之前的每个增量文件，直到找到一个完整文件。然后，构建一个需要从哪些源读取块的映射，并仅从每个源读取所需的块构成最后的目标文件，完成该过程的函数为:

```C
void
reconstruct_from_incremental_file(char *input_filename,         <--- 需要重构的文件名
								  char *output_filename,        <--- 目标文件名
								  char *relative_path,
								  char *bare_file_name,
								  int n_prior_backups,          <--- 备份个数
								  char **prior_backup_dirs,     <--- 备份目录
								  manifest_data **manifests,
								  char *manifest_path,
								  pg_checksum_type checksum_type,
								  int *checksum_length,
								  uint8 **checksum_payload,
								  bool debug,
								  bool dry_run)
```

具体的实现细节感兴趣请自行查看。


### 小结

以上就是 PostgreSQL 17 增量备份特性的介绍，WAL summarizer 生成记录页面变化的摘要文件，`pg_basebackup` 的增量备份模式生成增量备份，`pg_combinebackup` 则用于将全量备份和增量备份合并为一个新的全量备份。

希望读罢此文，你能对该特性有进一步的了解。另外，全量备份、增量备份和 WAL 归档仅仅是 PostgreSQL 提供的备份机制，具体的备份策略需要根据业务需求和实际情况进行制定。

### 题外话

增量备份的能力在一些外部备份工具中也有提供，如 [pgBackRest](https://github.com/pgbackrest/pgbackrest) 提供了 Full/Differential/Incremental 三种备份模式，为了避免概念上的混淆，我们了解一下 [pgBackRest 三种备份模式含义](https://pgbackrest.org/user-guide.html#concept/backup):

- Full Backup: pgBackRest 将整个数据库集群的内容复制到备份中，数据库集群的第一个备份始终是全量备份。
- Differential Backup: pgBackRest 仅复制自上一次全量备份以来发生变化的数据库集群文件。
- Incremental Backup: pgBackRest 仅复制自上一次备份（可以是另一个增量备份、差异备份或全量备份）以来发生变化的数据库集群文件。

[pg_rman](https://github.com/ossc-db/pg_rman) 支持全量备份和增量备份，timeline 变更之后需要先做一次全量备份再进行后续的增量备份:

- Full backup: Backup a whole database cluster.
- Incremental backup: Backup only files or pages modified after the last verified backup with the same timeline.

PostgreSQL 17 的增量备份不区分 Differenctial 和 Incremental，也没有 timeline 的限制，基于全量备份或增量备份所做的备份都叫增量备份。

另外 Greenplum 的恢复模式也用到了这三个词，为了进行区分，这里也简单介绍一下:

- Full Recovery: 全量修复，通过 pg_basebackup 的全量备份来实现
- Incremental Recovery: 增量修复，通过 pg_rewind 实现
- Differential Recovery: 差异化修复，基于 `rsync` 实现，我在之前的一篇文章中对此进行过分析: [GPDB 差异化修复原理简介](https://zhjwpku.com/2023/05/16/gpdb-differential-recovery-introduction.html)

可以看出，这些术语在 PG 生态不同的上下文（context，我非常喜欢的一个词）会被重复使用，希望这段题外话对你的理解有一定帮助。
