---
author: pg-x
title: "PITR 真的能恢复到任意时间点吗？"
date: 2023-09-29T11:29:00+08:00
tags: [PITR]
draft: false
ShowToc: false
TocOpen: false
---

国庆节前一个生产环境的备机 A2 在进行 pg_basebackup 时未加 `--write-recovery-conf` 且未创建 recovery.conf 文件后，手动将实例进行了启动（该环境本身由 patroni 控制 HA，但 patroni 一直未能将环境恢复，因此 DBA 进行了手动操作），导致 A2 和它原来的主 A1 以同样的 timeline 对外提供服务。这种情况下，我们还能将 A1 和 A2 恢复成主备关系吗？

首先由于 A1 和 A2 处于同一个 timeline，`pg_rewind` 会直接退出。一个同事提出：postgres 不是有 PITR 特性吗，能不能用 PITR 把 A2 恢复到上一个 timeline，然后再用 `pg_rewind` 将 A1 和 A2 恢复为主备关系？

### PITR 原理

PITR（Point-in-Time Recovery）是 PostgreSQL 数据库的一项重要功能，它允许你在数据库发生故障或数据丢失时恢复到特定的时间点。PITR 的工作原理是通过使用 WAL 日志来记录数据库的所有变更操作，从而实现对数据库状态的恢复。PITR 的第一步是创建数据库的基本备份，通常使用 PostgreSQL 的 pg_basebackup 工具来完成。这个基本备份是数据库的初始状态，用于在恢复过程中提供一个起点。

![Basic Concept of PITR](/images/pitr-basic-concept.png)

{{< rawhtml >}}
<h6 style="text-align: center;">图片来源: http://www.interdb.jp/pg/pgsql10.html</h6>
{{< /rawhtml >}}

如上图所示，PITR 需要以基本备份记录的 REDO point 为起点，回放之后的 WAL 日志。

### 回答题目的问题

单靠一个基本备份和归档日志 PITR 不能恢复到任意时间点，PITR 是通过 backup_label 中记录的 CHECKPOINT LOCATION 来找到对应的 REDO point，然后通过回放日志到 `recovery_target_time` 或 `recovery_target_lsn` 指定的位置。

当手动启动 A2 之后，其数据已经根据 WAL 日志进行了回放，甚至可能有新的写入，不借助另外的实例是不可能恢复到 base backup 完成时的状态的。值得一提的是 pg_rewind 是通过找到分叉点之前的 checkpoint，将之后 WAL 修改过的文件从源端拷贝到目的端进行覆盖，然后再回放日志来保证数据的一致性。

### PITR 最佳实践

当适用 PITR 功能时，以下操作能确保正确性和可靠性：

1. 定期备份数据库：进行定期的完整备份是 PITR 的基础。确保按照恢复点的需求和数据变更频率，制定合适的备份策略，比如每周或每月备份，并在备份过程中验证备份文件的完整性。
2. 设置可靠的归档日志目标：归档日志是 PITR 的关键组成部分。确保归档之日保存在一个独立、可靠的位置，比如 S3、NAS、RAID 等，同时确保具有足够的存储空间。
3. 配置自动归档：自动归档机制可以确保事务日志自动归档并保存到指定的目标位置。使用 PostgreSQL 的 archive_command 配置项来设置自动归档命令，并确保测试和验证归档过程的正确性。
4. 监控归档进程：监控归档进程的状态和性能是非常重要的。确保实时监控归档日志的生成和传输过程，以便及时发现和解决任何潜在的问题。
5. 定期验证恢复过程：定期进行 PITR 恢复测试是至关重要的。通过将数据库还原到不同的时间点，验证数据库的一致性和准确性。这有助于确保备份和归档机制的正常工作，并提前发现潜在的问题。
6. 清理过期的归档日志：定期清理过期的归档日志是维护 PITR 系统的必要步骤。根据备份策略和恢复点要求，确保及时删除不再需要的归档日志文件，以节省磁盘空间，并保持归档目录的整洁。

### 小结

PITR 需要通过 pg_basebackup 和 archive_command 配合才能保证恢复到预期的时间点，但数据量很大的时候做一次 pg_basebackup 耗时非常长，因此有些数据库厂商提供增量的基本备份，即当明确不需要一个备份时，通过增量备份的方式，将其变更为最新的基本备份，以减少磁盘 IO 和网络开销。
