---
author: zhjwpku
title: "UPDATE 语句 SET 索引列但未更改索引列值会走 HOT UPDATE 吗？"
date: 2024-08-10T13:40:31+08:00
tags: []
ShowToc: false
TocOpen: false
---

问题: PostgreSQL 中 UPDATE 操作 set 带索引的列，但值没变化还会走 HOT UPDATE 吗？

我们用 [PostgreSQL 14 internals](https://edu.postgrespro.com/postgresql_internals-14_en.pdf) 中的例子验证一下:

```SQL
CREATE TABLE accounts(
    id integer PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
    client text,
    amount numeric
);

INSERT INTO accounts VALUES
    (1, 'alice', 1000.00), (2, 'bob', 100.00), (3, 'bob', 900.00);
```

用 PostgreSQL 14 internals 中提供的两个 UDF（见附录）`heap_page` 和 `index_page` 查看 heap 表和 btree 的页面 layout:

```SQL
# SELECT * FROM heap_page('accounts', 0);
┌───────┬────────┬──────┬──────┬─────┬─────┬────────┐
│ ctid  │ state  │ xmin │ xmax │ hhu │ hot │ t_ctid │
├───────┼────────┼──────┼──────┼─────┼─────┼────────┤
│ (0,1) │ normal │ 920  │ 0 a  │     │     │ (0,1)  │
│ (0,2) │ normal │ 920  │ 0 a  │     │     │ (0,2)  │
│ (0,3) │ normal │ 920  │ 0 a  │     │     │ (0,3)  │
└───────┴────────┴──────┴──────┴─────┴─────┴────────┘
(3 rows)

# SELECT * FROM index_page('accounts_pkey', 1);
┌────────────┬───────┬──────┐
│ itemoffset │ htid  │ dead │
├────────────┼───────┼──────┤
│          1 │ (0,1) │ f    │
│          2 │ (0,2) │ f    │
│          3 │ (0,3) │ f    │
└────────────┴───────┴──────┘
(3 rows)
```

然后进行如下更新语句，该语句 set 索引列但不更新索引列值:

```SQL
UPDATE accounts SET id = 1, amount = 2000.00 WHERE id = 1;
```

再次查看 layout:

```SQL
# SELECT * FROM heap_page('accounts', 0);
┌───────┬────────┬───────┬──────┬─────┬─────┬────────┐
│ ctid  │ state  │ xmin  │ xmax │ hhu │ hot │ t_ctid │
├───────┼────────┼───────┼──────┼─────┼─────┼────────┤
│ (0,1) │ normal │ 920 c │ 921  │ t   │     │ (0,4)  │
│ (0,2) │ normal │ 920   │ 0 a  │     │     │ (0,2)  │
│ (0,3) │ normal │ 920   │ 0 a  │     │     │ (0,3)  │
│ (0,4) │ normal │ 921   │ 0 a  │     │ t   │ (0,4)  │
└───────┴────────┴───────┴──────┴─────┴─────┴────────┘
(4 rows)

SELECT * FROM index_page('accounts_pkey', 1);
┌────────────┬───────┬──────┐
│ itemoffset │ htid  │ dead │
├────────────┼───────┼──────┤
│          1 │ (0,1) │ f    │
│          2 │ (0,2) │ f    │
│          3 │ (0,3) │ f    │
└────────────┴───────┴──────┘
(3 rows)

```

可以看到 (0,1) 元组指向了 (0,4)，索引上对应的 entry 依旧指向 (0,1)，并且元组的头信息设置了标记位 HEAP_HOT_UPDATED 和 HEAP_ONLY_TUPLE。

```C
/*
 * information stored in t_infomask2:
 */
#define HEAP_NATTS_MASK			0x07FF	/* 11 bits for number of attributes */
/* bits 0x1800 are available */
#define HEAP_KEYS_UPDATED		0x2000	/* tuple was updated and key cols
										 * modified, or tuple deleted */
#define HEAP_HOT_UPDATED		0x4000	/* tuple was HOT-updated */
#define HEAP_ONLY_TUPLE			0x8000	/* this is heap-only tuple */

#define HEAP2_XACT_MASK			0xE000	/* visibility-related bits */
```

上面的页面结构还有一个信息，(0,1) 元组的状态是 normal，这是因为此时该页面还未进行过 page pruning 或 vacuum，其数据依然存在，对其进行 vacuum 后，元组内容会被清除，itempointer 的状态变为 redirect。

```SQL
# vacuum accounts;
VACUUM

# SELECT * FROM heap_page('accounts', 0);
┌───────┬───────────────┬───────┬──────┬─────┬─────┬────────┐
│ ctid  │     state     │ xmin  │ xmax │ hhu │ hot │ t_ctid │
├───────┼───────────────┼───────┼──────┼─────┼─────┼────────┤
│ (0,1) │ redirect to 4 │       │      │     │     │        │
│ (0,2) │ normal        │ 920 c │ 0 a  │     │     │ (0,2)  │
│ (0,3) │ normal        │ 920 c │ 0 a  │     │     │ (0,3)  │
│ (0,4) │ normal        │ 921 c │ 0 a  │     │ t   │ (0,4)  │
└───────┴───────────────┴───────┴──────┴─────┴─────┴────────┘
(4 rows)
```

### 源码分析

从代码层面看下为什么上面的语句能够走 HOT UPDATE 逻辑，首先，更新操作执行的算子是 `ModifyTable`，执行 update 首先会调用 `ExecGetUpdateNewTuple` 准备一个新插入的元组，然后执行 `ExecUpdate`:

```C
	slot = ExecGetUpdateNewTuple(resultRelInfo, context.planSlot,
								 oldSlot);

	/* Now apply the update. */
	slot = ExecUpdate(&context, resultRelInfo, tupleid, oldtuple,
					  slot, node->canSetTag);
```

从 `ExecUpdate` 一路跟下去（`ExecUpdateAct` -> `table_tuple_update` -> `heapam_tuple_update` -> `heap_update`），在 heap_update 中调用 `HeapDetermineColumnsInfo` 对比更新前后的元组中真正修改的列并设置 bitmap:

```C
		/*
		 * Extract the corresponding values.  XXX this is pretty inefficient
		 * if there are many indexed columns.  Should we do a single
		 * heap_deform_tuple call on each tuple, instead?	But that doesn't
		 * work for system columns ...
		 */
		value1 = heap_getattr(oldtup, attrnum, tupdesc, &isnull1);
		value2 = heap_getattr(newtup, attrnum, tupdesc, &isnull2);

		if (!heap_attr_equals(tupdesc, attrnum, value1,
							  value2, isnull1, isnull2))
		{
			modified = bms_add_member(modified, attidx);
			continue;
		}
```

对于上述的更新语句，虽然 SET 了索引列，但其值并没有真正被修改，所以它并不会囊括到生成的 bitmap 中，`heap_update` 中如下逻辑是成立的:

```C
	/*
	 * At this point newbuf and buffer are both pinned and locked, and newbuf
	 * has enough space for the new tuple.  If they are the same buffer, only
	 * one pin is held.
	 */

	if (newbuf == buffer)
	{
		/*
		 * Since the new tuple is going into the same page, we might be able
		 * to do a HOT update.  Check if any of the index columns have been
		 * changed.
		 */
		if (!bms_overlap(modified_attrs, hot_attrs))
		{
			use_hot_update = true;
```

因此 HOT UPDATE 得以继续使用。另外，如果使用了 HOT UPDATE，`heap_update` 的出参 **update_indexes** 会被设置为 TU_None 或 TU_Summarizing（BRIN 相关，未深入研究），即索引无需更新。

```C
	/*
	 * If it is a HOT update, the update may still need to update summarized
	 * indexes, lest we fail to update those summaries and get incorrect
	 * results (for example, minmax bounds of the block may change with this
	 * update).
	 */
	if (use_hot_update)
	{
		if (summarized_update)
			*update_indexes = TU_Summarizing;
		else
			*update_indexes = TU_None;
	}
	else
		*update_indexes = TU_All;
```

### Appendix

```SQL
DROP FUNCTION IF EXISTS index_page(text, integer);
CREATE FUNCTION index_page(relname text, pageno integer)
    RETURNS TABLE(itemoffset smallint, htid tid, dead boolean)
AS $$
    SELECT itemoffset,
           htid,
           dead -- starting from v.13
    FROM bt_page_items(relname,pageno);
$$ LANGUAGE sql;

DROP FUNCTION IF EXISTS heap_page(text,integer);
CREATE FUNCTION heap_page(relname text, pageno integer)
    RETURNS TABLE(
        ctid tid, state text,
        xmin text, xmax text,
        hhu text, hot text, t_ctid tid)
AS $$
    SELECT (pageno,lp)::text::tid AS ctid,
        CASE lp_flags
          WHEN 0 THEN 'unused'
          WHEN 1 THEN 'normal'
          WHEN 2 THEN 'redirect to '||lp_off
          WHEN 3 THEN 'dead'
        END AS state,
        t_xmin || CASE
          WHEN (t_infomask & 256) > 0 THEN ' c'
          WHEN (t_infomask & 512) > 0 THEN ' a'
          ELSE ''
        END AS xmin,
        t_xmax || CASE
          WHEN (t_infomask & 1024) > 0 THEN ' c'
          WHEN (t_infomask & 2048) > 0 THEN ' a'
          ELSE ''
        END AS xmax,
        CASE WHEN (t_infomask2 & 16384) > 0 THEN 't' END AS hhu,
        CASE WHEN (t_infomask2 & 32768) > 0 THEN 't' END AS hot,
        t_ctid
    FROM heap_page_items(get_raw_page(relname,pageno))
    ORDER BY lp;
$$ LANGUAGE sql;
```