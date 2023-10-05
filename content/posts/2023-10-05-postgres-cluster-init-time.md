---
author: pg-x
title: "如何获取 PostgreSQL 实例的创建时间？"
date: 2023-10-05T09:33:36+08:00
tags: []
draft: false
ShowToc: false
TocOpen: false
---

PostgreSQL 实例在启动的时候会创建一个 `postmaster.pid` 文件，通过查看该文件的状态可以知道实例启动的时间，另外 pg_stat_activity 记录了辅助进程的状态，因此我们可以通过 `select pg_stat_file('postmaster.pid') AS start_time;` 或 `select MIN(backend_start) as start_time from pg_stat_activity;` 来获取实例启动的时间，但如何知道实例的创建时间呢？

PostgreSQL 创建实例（initdb）的时候会写一个 `global/pg_control` 文件，这其中有一个字段 **Database system identifier**，在实例 bootstrap 的时候初始化该字段，且在实例之后的生命周期中不再变更。

```C
void
BootStrapXLOG(void)
{
    ...

	gettimeofday(&tv, NULL);
	sysidentifier = ((uint64) tv.tv_sec) << 32;
	sysidentifier |= ((uint64) tv.tv_usec) << 12;
	sysidentifier |= getpid() & 0xFFF;

    ...

    /* Now create pg_control */
	InitControlFile(sysidentifier);

    ...
}
```

因此我们可以读取该字段，解析其高 32 位即可获取实例创建的时间：

```shell
➜  postgres_data pg_controldata . | head
pg_control version number:            1300
Catalog version number:               202308241
Database system identifier:           7271454612807718361
Database cluster state:               in production
pg_control last modified:             Mon Oct  2 12:56:26 2023
Latest checkpoint location:           0/2C0163A0
Latest checkpoint's REDO location:    0/2C016368
Latest checkpoint's REDO WAL file:    00000001000000000000002C
Latest checkpoint's TimeLineID:       1
Latest checkpoint's PrevTimeLineID:   1
```

```SQL
postgres=# select to_hex(7271454612807718361);
      to_hex
------------------
 64e96571ccc795d9
(1 row)

postgres=# select '0x64e96571'::int;
    int4
------------
 1693017457
(1 row)

postgres=# select to_timestamp(1693017457) as cluster_init;
      to_timestamp
------------------------
 2023-08-26 10:37:37+08
(1 row)

```

或者用一条语句获取实例创建时间:

```SQL
postgres=# select to_timestamp ( system_identifier >> 32 ) as cluster_init from pg_control_system();
      cluster_init
------------------------
 2023-08-26 10:37:37+08
(1 row)
```
