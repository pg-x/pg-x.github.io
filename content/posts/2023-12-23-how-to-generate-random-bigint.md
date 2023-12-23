---
author: pg-x
title: "PostgreSQL 生成随机 bigint"
date: 2023-12-23T10:31:54+08:00
tags: []
ShowToc: false
TocOpen: false
---

PostgreSQL 的 `random()` 函数用于生成一个随机的双精度浮点数，可以将其转换为较小的随机整数，但是双精度浮点数无法表示 bigint 中的所有值[^1]，本文介绍几种 PostgreSQL 生成 bigint 随机数的方法。

### UUID

我们可以提取 `gen_random_uuid` 生成的随机位将其转换为 bigint，如:

```SQL
postgres=# select ('x'||right(uuid_send(gen_random_uuid())::text, 16))::bit(64)::int8;
         int8
----------------------
 -8048312917378578936
(1 row)
```

但了解 UUIDv4 的同学一眼就能看出问题，虽然 UUIDv4 中的 122 位都是随机数，但上述方法使用的后 64 位中的前两位一定是 `0b10`（参考: [PostgreSQL 引入 UUIDv7](/posts/2023-12-09-postgres-introduce-uuidv7/)），因此上述方法生成的随机 bigint 一定是个负数。所以我们需要从 UUIDv4 中截取两段数据才能生成一个随机的 bigint:

```SQL
postgres=# CREATE OR REPLACE FUNCTION gen_random_bigint_from_uuid() RETURNS INT8 AS $$
DECLARE
    bytes text;
BEGIN
    bytes := uuid_send(gen_random_uuid())::text;
    RETURN ('x' || substring(bytes, 3, 2) || right(bytes, 14))::bit(64)::int8;
END;
$$ LANGUAGE plpgsql;
CREATE FUNCTION
postgres=# select gen_random_bigint_from_uuid() from generate_series(1,5);
 gen_random_bigint_from_uuid
-----------------------------
         2561119870940783961
        -8836220962687819417
        -5760399065689643664
        -8861236575235818608
         7084792777938647791
(5 rows)
```

注意 UUIDv7 因为只有 62 位的随机值，因此不能直接用来生成 bigint 随机值，不过可以结合 `random()` 函数进行拼接，此不详述。

### pg_random_bytes

内置插件 **pgcrypto** 提供的 `pg_random_bytes()` 函数可以生成随机 bytea，可以将此拼接为随机 bigint:

```SQL
CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE OR REPLACE FUNCTION gen_random_int8() RETURNS INT8 AS $$
DECLARE
    bytes bytea;
BEGIN
    bytes := gen_random_bytes(8);
RETURN
    (get_byte(bytes,0)::int8 << 8*0) |
    (get_byte(bytes,1)::int8 << 8*1) |
    (get_byte(bytes,2)::int8 << 8*2) |
    (get_byte(bytes,3)::int8 << 8*3) |
    (get_byte(bytes,4)::int8 << 8*4) |
    (get_byte(bytes,5)::int8 << 8*5) |
    (get_byte(bytes,6)::int8 << 8*6) |
    (get_byte(bytes,7)::int8 << 8*7);
END;
$$ LANGUAGE plpgsql;
```

或者直接用 `encode` 函数将其编码为 hex 再显示转为 bigint:

```SQL
postgres=# select ('x'||encode(gen_random_bytes(8), 'hex'))::bit(64)::int8 from generate_series(1,5);
         int8
----------------------
 -8930476623996945156
 -5126984886511757661
 -6072786556490332588
  3278019797672496176
 -2060333909321021593
(5 rows)
```

### /dev/urandom

`/dev/urandom` 是一个在类 Unix 系统中用于生成高质量伪随机数据的特殊文件，常用于各种安全应用和加密操作。PostgreSQL 提供的 `pg_read_binary_file` 可以从该文件中读取随机数据，进而转换成随机 bigint:

```SQL
postgres=# select ('x'||encode(pg_read_binary_file('/dev/urandom', 0, 8), 'hex'))::bit(64)::bigint from generate_series(1, 5);
         int8
----------------------
  1349504348694981204
  4053722996198320118
  7178451800259674372
 -1207924631398290525
  3857806568552511469
(5 rows)
```

### random()

既然 `random()` 能完整表示 32 位整数，那将两次 `random()` 获得的 32 位整数进行拼接自然可以获得一个随机的 64 位整数:

```SQL
postgres=# select ((random() * 4294967296)::int8::bit(32) || (random() * 4294967296)::int8::bit(32))::bit(64)::int8 from generate_series(1,5);
         int8
----------------------
 -6168096176574416042
  8362256655007089958
  4100237736368010094
  5317107880510500415
  7978814854907299951
(5 rows)
```

### 性能对比

本文介绍了 5 种生成随机 bigint 值的方法，对这五种方法的执行效率做一个对比:

```SQL
postgres=# explain analyze select gen_random_bigint_from_uuid() from generate_series(1, 1000000);
Time: 6097.280 ms (00:06.097)
postgres=# explain analyze select gen_random_int8() from generate_series(1, 1000000);
Time: 4712.437 ms (00:04.712)
postgres=# explain analyze select ('x'||encode(gen_random_bytes(8), 'hex'))::bit(64)::int8 from generate_series(1, 1000000);
Time: 3080.222 ms (00:03.080)
postgres=# explain analyze select ('x'||encode(pg_read_binary_file('/dev/urandom', 0, 8), 'hex'))::bit(64)::bigint from generate_series(1, 1000000);
Time: 19539.961 ms (00:19.540)
postgres=# explain analyze select ((random() * 4294967296)::int8::bit(32) || (random() * 4294967296)::int8::bit(32))::bit(64)::int8 from generate_series(1, 1000000);
Time: 1170.033 ms (00:01.170)
```

可见，将两个 `random()` 生成的值进行拼接生成 bigint 的性能最好。

本文灵感来自 pg-general 邮件列表的问题：[How to generate random bigint](https://www.postgresql.org/message-id/flat/CAGAwPgQt6B0ORW_YfwsdRmWPhb_pG6c9HM7S=G4WGC_s8qaU+w@mail.gmail.com)


[^1]: https://stackoverflow.com/a/43656339
