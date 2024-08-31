---
author: zhjwpku
title: "pg_squeeze vs. VACUUM FULL CONCURRENTLY"
date: 2024-08-31T10:11:30+08:00
tags: ["pg_squeeze", "vacuum"]
TocOpen: false
---

[pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze) 是 [CYBERTEC](https://www.cybertec-postgresql.com/) 开源的一个 PostgreSQL 扩展，提供了一个后台工作进程（每个数据库一个），定期监控用户定义的表，当检测到某个表超出了**膨胀阈值**时，`pg_squeeze` 会自动开始重建该表。重建过程在后台并发进行，它利用 PostgreSQL 内置的复制槽和 logical decoding，从 WAL 中提取重建期间发生的表更改。pg_squeeze 要求重建的表必须定义**主键**或**唯一约束**。

使用 pg_squeeze 需要在 `postgresql.conf` 中作相应的配置:

```txt
wal_level = logical                         # pg_squeeze 使用 logical decoding
shared_preload_libraries = 'pg_squeeze'     # 用到了共享内存和 BackgroundWorker
```

用户可以通过将表的信息插入 `squeeze.tables` 表中来注册表。注册后，系统会定期检查表的状态，当满足条件时，会自动触发重建过程。对于不想注册表或者不需要预先检查膨胀的情况，可以手动重建。

```SQL
-- 1. Register table for regular processing
INSERT INTO squeeze.tables (
    tabschema,
    tabname,
    schedule,
    free_space_extra,
    vacuum_max_age,
    max_retry
)
VALUES (
    'public',
    'bar',
    ('{30}', '{22}', NULL, NULL, '{3, 5}'),
    30,
    '2 hours',
    2
);

-- 2. Ad-hoc processing for any table
SELECT squeeze.squeeze_table('public', 'foo', 'foo_i_idx');
```

第一种方式（入口函数: `scheduler_worker_loop`）依赖 pg_squeeze 内部实现的一个调度框架（scheduler worker 定期轮询并生成要执行的任务，并调度到空闲的 squeeze worker 上去执行），第二种方式（入口函数: `squeeze_table_new`）则是直接启动一个 squeeze worker 来执行重建任务。

不论哪种方式，对于一个重建任务，最终都会调用 `squeeze_table_internal` 这个函数，该函数的几个关键步骤:

```C
static void
squeeze_table_internal(Name relschema, Name relname, Name indname,
					   Name tbspname, ArrayType *ind_tbsp)
{
    ...
    // 设置 logical decoding 所需的上下文，包括从之前创建好的 replication slot 中反序列化
    // snapshot，创建 decoding context，创建 tuplestore 等
    ctx = setup_decoding(relid_src, tup_desc, &snap_hist);

    // 创建目的表
    relid_dst = create_transient_table(cat_state, tup_desc, tbsp_info->table,
									   rel_src_owner);

    // 打开源表，使用的共享锁，该表依然可以进行 DML 操作
    rel_src = table_open(relid_src, AccessShareLock);

    // 打开目的表，目的表使用的是排他锁
    rel_dst = table_open(relid_dst, AccessExclusiveLock);

    // 根据上面获取的 snapshot 将源表数据插入到目的表，可以根据设置的索引进行重排序
    perform_initial_load(rel_src, relrv_cl_idx, snap_hist, rel_dst, ctx);

    // 在临时的目的表上创建索引
    indexes_dst = build_transient_indexes(rel_dst, rel_src, indexes_src,
										  nindexes, tbsp_info, cat_state,
										  ctx);

    // 构建将在逻辑解码过程中用于查找要更新或删除的行的扫描键
    ident_key = build_identity_key(ident_idx_src, rel_src,
								   &ident_key_nentries);

    // 释放共享锁，后面的 perform_final_merge 会对源表加排他锁
    table_close(rel_src, AccessShareLock);

    // 将到目前为止插入的所有WAL（Write-Ahead Logging）记录刷新到磁盘，
    // 以最小化在持有源表排他锁时需要刷新的数据量
    xlog_insert_ptr = GetInsertRecPtr();
	XLogFlush(xlog_insert_ptr);

    // 解码并回放在 initial load 进行时发生的数据变更。XLOG读取器应该继续从
    // setup_decoding 函数停止的地方开始处理
    process_concurrent_changes(ctx, end_of_wal, cat_state, rel_dst,
							   ident_key, ident_key_nentries, iistate,
							   NoLock, NULL);

    // 尝试对源表的并发数据变更执行最终处理
    source_finalized = false;
	for (i = 0; i < 4; i++)
	{
        // 该函数内部会加排他锁
		if (perform_final_merge(relid_src, indexes_src, nindexes,
								rel_dst, ident_key, ident_key_nentries,
								iistate, cat_state, ctx))
		{
			source_finalized = true;
			break;
		}
		else
			elog(DEBUG1,
				 "pg_squeeze: exclusive lock on table %u had to be released.",
				 relid_src);
	}

    // 清理 logical decoding 上下文
    decoding_cleanup(ctx);

    table_close(rel_dst, AccessExclusiveLock);

    // 在系统表交换源表和目标表相关的文件 oid
    // 包含 relfilenode/reltablespace/reltoastrelid
    swap_relation_files(relid_src, relid_dst);

    // 交换 toast 表名称
    swap_toast_names(relid_src, toastrelid_dst, relid_dst, toastrelid_src);

    // 交换索引相关的 oid
    for (i = 0; i < nindexes; i++)
		swap_relation_files(indexes_src[i], indexes_dst[i]);

    // 删除源表相关的对象（这里删除 relid_dst 是因为上面进行了交换）
    object.classId = RelationRelationId;
	object.objectSubId = 0;
	object.objectId = relid_dst;
	performDeletion(&object, DROP_RESTRICT, PERFORM_DELETION_INTERNAL);
}
```

看完上面的代码注释，相信读者对 pg_squeeze 的单表重建逻辑会有一个大概的了解。我认为最关键的三个步骤:

- `perform_initial_load` 根据快照将源表数据拷贝到目标表
- `process_concurrent_changes` 处理第一步过程中同时发生的 DML 操作
- `perform_final_merge` 这一步会加排他锁，使用 **process_concurrent_changes** 处理第二步过程中发生的 DML 操作

其中还有很多细节，比如同步过程中检查重建表的 catalog 有没有变更、如果有过多的 DML 是否需要超时处理、initial load 之后需要先建立索引再去处理 DML 操作等，感兴趣的同学可以查看相关的源码。

pg_squeeze 使用 `CreateInitDecodingContext` 把动态链接库的名字（`#define	REPL_PLUGIN_NAME	"pg_squeeze"`） 作为参数传递下去，`LoadOutputPlugin` 初始化 Logical Decoding 约定的函数 `_PG_output_plugin_init` 设置需要的回调函数。这是写 Logical Decoding 插件固有的模式，如果第一次接触，可能读起来不是那么直观，这里特意说明一下。

### VACUUM FULL CONCURRENTLY

pg_squeeze 的单表重建逻辑相较于 `VACUUM FULL` 的优势在于它仅在短期内对表施加了排他锁，从而最大程度地减少了对源表可用性的影响。

那为什么 PostgreSQL 自己不提供这样的功能呢？比如提供一个 `VACUUM FULL CONCURRENTLY` 命令？

在今年（2024）一月，pgsql-hackers 邮件列表有人问了这样一个问题，[why there is not VACUUM FULL CONCURRENTLY?](https://www.postgresql.org/message-id/CAFj8pRDK89FtY_yyGw7-MW-zTaHOCY4m6qfLRittdoPocz+dMQ@mail.gmail.com)

```TXT
Hi

I have one question, what is a block of implementation of some variant of
VACUUM FULL like REINDEX CONCURRENTLY? Why similar mechanism of REINDEX
CONCURRENTLY cannot be used for VACUUM FULL?
```

其实类似的问题在 2012 年的 [pg_reorg in core?](https://www.postgresql.org/message-id/CAB7nPqTGmNUFi%2BW6F1iwmf7J-o6sY%2Bxxo6Yb%3DmkUVYT-CG-B5A%40mail.gmail.com) 已经有过讨论，[pg_reorg](https://github.com/reorg/pg_reorg) 是 [pg_repack](https://github.com/reorg/pg_repack) 的前身，同样提供不长期持有排他锁的前提下移除表和索引膨胀，与 pg_squeeze 不同的是它使用了**触发器**来将重建过程产生的 DML 保存在一个临时表中。

如今的 logical decoding 已经非常成熟，Alvaro Herrera 提议直接把 pg_squeeze 的逻辑集成到 VACUUM 命令中:

```txt
FWIW a newer, more modern and more trustworthy alternative to pg_repack
is pg_squeeze, which I discovered almost by random chance, and soon
discovered I liked it much more.

So thinking about your question, I think it might be possible to
integrate a tool that works like pg_squeeze, such that it runs when
VACUUM is invoked -- either under some new option, or just replace the
code under FULL, not sure.  If the Cybertec people allows it, we could
just grab the pg_squeeze code and add it to the things that VACUUM can
run.

Now, pg_squeeze has some additional features, such as periodic
"squeezing" of tables.  In a first attempt, for simplicity, I would
leave that stuff out and just allow it to run from the user invoking it,
and then have the command to do a single run.  (The scheduling features
could be added later, or somehow integrated into autovacuum, or maybe
something else.)
```

CYBERTEC pg_squeeze 的负责人迅速做出了响应并给出了 in-core 的实现，目前 patch 还在 review 中，如果这个功能能够集成到 PostgreSQL 18，将为用户带来更大的便利。

```TXT
The necessity of reducing table size is not too common (a lot of use cases 
are better covered by using partitioning), but sometimes it is, and then 
buildin simple available solution can be helpful.
```


### References

- [pg_squeeze: Optimizing PostgreSQL storage](https://www.cybertec-postgresql.com/en/pg_squeeze-optimizing-postgresql-storage/)
- [Introducing pg_squeeze - a PostgreSQL extension to auto-rebuild bloated tables](https://www.cybertec-postgresql.com/en/introducing-pg_squeeze-a-postgresql-extension-to-auto-rebuild-bloated-tables/)
