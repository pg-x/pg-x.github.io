---
author: pg-x
title: "pg_cron 支持 Last Day of Month 调度"
date: 2023-08-22T17:15:06+08:00
draft: false
tags: ["pg_cron"]
ShowToc: false
TocOpen: false
---

[pg_cron](https://github.com/citusdata/pg_cron) 是 PostgreSQL 数据库的一个插件，用于在 Postgres 数据库中实现定时任务功能。它内嵌了 [vixie's cron](https://github.com/vixie/cron) 作为调度条件的判断，不同于 cron 使用 crontab 文件来保存任务信息，pg_cron 任务详情保存在表 cron.job 中，一个名为 `pg_cron scheduler` 的 background worker 周期性地读取 cron.job 的任务列表并将其更新到 pg_cron 的内存缓存结构中，然后判断每条任务是否满足执行条件，如果满足则调度任务执行。

### 常见用法

```SQL
-- 匿名任务: 定时删除过期数据，在周六凌晨三点半删除一周前的数据
SELECT cron.schedule('30 3 * * 6', $$DELETE FROM events WHERE event_time < now() - interval '1 week'$$);

-- 具名任务: 在每天早上10点对插件所在的 database 执行 vaccum
SELECT cron.schedule('nightly-vacuum', '0 10 * * *', 'VACUUM');

-- 根据任务名称更新调度任务，将调度改为凌晨3点
SELECT cron.schedule('nightly-vacuum', '0 3 * * *', 'VACUUM');

-- 每隔 5 秒钟执行一次名为 process_updates 的存储过程
SELECT cron.schedule('process-updates', '5 seconds', 'CALL process_updates()');
```

上述涉及到定时任务的时间都为 GMT 时区，如需更改时区需要设置 'cron.timezone' 这个 GUC。关于 cron 调度的表达式可参考 [crontab guru](https://crontab.guru/)。

### Last day of month

对于标准的 cron 任务，如果想要实现在每个月的最后一天去调度一个任务，可能会有一些麻烦，比如可以使用如下语句:

```shell
30 3 28-31 * * [ "$(date +\%m -d tomorrow)" != "$(date +\%m)" ] && real-command
```

上面的调度在 28-31 号的凌晨3点半都会去执行，如果当天和第二天不属于同一个月，则证明是当月的最后一天，然后执行真正的命令。另一种方式是通过设置多条任务来模拟，比如在每个月的最后一天发工资可以用如下语句实现:

```SQL
-- 二月28号中午12点
SELECT cron.schedule('0 12 28 2 *', 'CALL pay_salary()');

-- 1,3,5,7,8,10,12 月的31号中午12点
SELECT cron.schedule('0 12 31 1,3,5,7,8,10,12 *', 'CALL pay_salary()');

-- 2,6,9,11 月的30号中午12点
SELECT cron.schedule('0 12 30 2,6,9,11 *', 'CALL pay_salary()');
```

但这其中有一个小问题，闰年的二月最后一天是 29 号，上面的方法显然不够优雅。因此有些 cron 的实现增加了 last day of month 调度，比如 [AmazonCloudWatch](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/ScheduledEvents.html) 使用 `L` 来表示当月的最后一天。

### 实现细节

[vixie's cron](https://github.com/vixie/cron) 的一个 [issue](https://github.com/vixie/cron/issues/4) 建议使用 `day-of-the-month` 字段的第 0 位表示上个月的最后一天，但这看上去很奇怪，实现起来也有点不合常理。最终的实现用一个标志位（DOM_LAST）来表示一个任务是否需要在指定月的最后一天调度，在 `find_jobs` 函数中根据当前时间判断第二天的 tm_mday 为 1 ，且 `(e->flags & DOM_LAST) != 0`，则将任务加入到待执行队列。

在创建任务的时候，使用 `$` 表示最后一天，类比 vim 用 `$` 快速定位到一行的最后，cron 用 `$` 表示 last day of the month 可能也有一些程序员的情怀在里边。PR 见 [allow '$' to indicate last day-of-month](https://github.com/vixie/cron/pull/20)。

在 [vixie's cron](https://github.com/vixie/cron) 合入这个特性之后，迁移到 pg_cron 就非常简单了，对 `parse_cron_entry` 和 `ShouldRunTask` 两个函数进行了一些改造，具体实现见 [add possibility to schedule jobs on the last day of month](https://github.com/citusdata/pg_cron/pull/273)。


### 使用场景

有了这个特性之后，在每个月最后一天发薪的功能就很容易实现了:

```SQL
SELECT cron.schedule('0 12 $ * *', 'CALL pay_salary()');
```

列举一些其它的应用场景:

- 在每个月的最后一天创建下个月的分区表
- 在每个月的最后一天提醒你该立下个月的 flag 了
- 在每个月的最后一天该给爸妈打个电话问候一下了
- ...

### 小结

`pg_cron` 的 last-day-of-month 特性在 2023/8/21 日刚刚合入，感兴趣的小伙伴可以尝试一下。另外留一个思考题: 每个月的最后一个工作日应该怎么实现呢？ 🧐
