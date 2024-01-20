---
author: pg-x
title: "PostgreSQL COPY ON_ERROR"
date: 2024-01-20T15:56:06+08:00
tags: ["Waiting for PG17"]
ShowToc: false
TocOpen: false
---

PostgreSQL 17 对 COPY 命令做了一个优化，导入数据（COPY FROM）时可以忽略错误字段进而使得 COPY 命令继续导入后面的数据。虽然是很小的一个特性，但非常实用。

```shell
cat << EOF > /tmp/malformed_data.txt
 1	{1}	1
 a	{2}	2
 3	{3}	3333333333
 4	{a, 4}	4
 5	{5}	5
EOF
```

之前在用 COPY 导入数据时，如果文件中有不符合类型的字段，命令直接报错退出:

```SQL
postgres=# CREATE TABLE check_ign_err (n int, m int[], k int);
CREATE TABLE
postgres=# copy check_ign_err from '/tmp/malformed_data.txt';
ERROR:  invalid input syntax for type integer: " a"
CONTEXT:  COPY check_ign_err, line 2, column n: " a"
postgres=# table check_ign_err;
 n | m | k
---+---+---
(0 rows)
```

PostgreSQL 17 增加了如下语法:

```SQL
postgres=# copy check_ign_err from '/tmp/malformed_data.txt' (ON_ERROR stop);
ERROR:  invalid input syntax for type integer: " a"
CONTEXT:  COPY check_ign_err, line 2, column n: " a"
postgres=# copy check_ign_err from '/tmp/malformed_data.txt' (ON_ERROR ignore);
NOTICE:  3 rows were skipped due to data type incompatibility
COPY 2
postgres=# table check_ign_err;
 n |  m  | k
---+-----+---
 1 | {1} | 1
 5 | {5} | 5
(2 rows)
```

新增加的语法 ON_ERROR 目前只有 `stop` 和 `ignore` 两种模式，stop 跟不带 ON_ERROR 的行为是一致的，`ignore` 则忽略错误的记录，继续执行，上面的例子显示了正常数据被成功保存到了表中，并给出了总共出错的数据行数。未来可能还会增加 `file 'filename.log'` 将错误信息保存到文件中或 `table 'tablename'` 将错误信息保存到表中，能够更方便地查看文件出错的位置。

> The option names now are "stop" (default) and "ignore".  The future options could be "file 'filename.log'" and "table 'tablename'".

Greenplum 也提供类似的功能，对于 OLAP 场景，数据量更大，如果文件导入后期出现错误，失败代价更大。其语法及使用方式跟 PostgreSQL 有所不同: `LOG ERRORS [ SEGMENT REJECT LIMIT <count> [ ROWS | PERCENT ] ]`，具体请参考 GPDB 的手册。
