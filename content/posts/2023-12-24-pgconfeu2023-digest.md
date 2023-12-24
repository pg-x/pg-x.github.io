---
author: pg-x
title: "PGConf.EU 2023 Digest"
date: 2023-12-24T08:55:47+08:00
tags: []
ShowToc: false
TocOpen: false
---

PGConf Europe 是欧洲最大的 PG 会议，[PGConf.EU 2023](https://2023.pgconf.eu/) 于 12 月 12-15 日在捷克首都布拉格举办。本文摘录一些我感兴趣的议题。

会议的时间表: [https://www.postgresql.eu/events/pgconfeu2023/schedule/](https://www.postgresql.eu/events/pgconfeu2023/schedule/)

### General

- **Elephant in a nutshell - Navigating the Postgres community 101**
    - 介绍 PostgreSQL 社区的运作机制
    - Speaker: Valeria Kaplan
    - Slides: [PGconf.EU. 2023_Valeria's talk - Elephant in a nutshell.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4855/slides/451/PGconf.EU.%202023_Valeria's%20talk%20-%20Elephant%20in%20a%20nutshell.pdf)


### DBA

- **Getting the most out of pg_stat_io**
    - PG16 引入了 pg_stat_io 视图，提供了很多 I/O 相关的细节，包括扩展、读取和写入，以及与缓冲区缓存相关的统计信息。
    - Speaker: Daniel Westermann
    - Slides: [PGCONFEU-pg_stat_io.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4774/slides/402/P-DBI-E-20231216-PGCONFEU-pg_stat_io.pdf)

- **Performance tricks you have never seen before-V2**
    - Speaker: Hans-Jürgen Schönig
    - Slides: [Performance tips you have never seen before - V2 (1).pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4706/slides/437/Performance%20tips%20you%20have%20never%20seen%20before%20-%20V2%20(1).pdf)

- **The journey towards active-active replication in PostgreSQL**
    - 介绍了逻辑复制的发展历程及 active-active 复制相关的特性支持
    - Speaker: Jonathan S. Katz
    - Slides: [pgconfeu2023_active_active.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4783/slides/434/pgconfeu2023_active_active.pdf)

- **Understanding and Fixing Autovacuum**
    - 介绍 autovacuum 常见的问题及解决方案。
    - Speaker: Robert Haas
    - Slides: [Understanding and Fixing Autovacuum - PGCONF.EU 2023.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4847/slides/432/Understanding%20and%20Fixing%20Autovacuum%20-%20PGCONF.EU%202023.pdf)

- **PostgreSQL Distributed: Architectures & Best practices**
    - 介绍了对分布式数据库的一些思考
    - Speaker: Marco Slot
    - Slides: [Distributed PostgreSQL 2023.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4826/slides/416/Distributed%20PostgreSQL%202023.pdf)

- **How to corrupt your database (and how to deal with data corruption)**
    - 如果你知道了如何搞崩你的数据库，你自然就知道如何避免让你的数据库崩溃。
    - Speaker: Laurenz Albe
    - Slides: [data_corruption.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4602/slides/406/data_corruption.pdf)

- **PostgreSQL Security: The defense line for your data**
    - Speaker: Julia Gugel
    - Slides: [PostgreSQL_security_with_demo.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4707/slides/444/P-DBI-E-20231214-PostgreSQL_security_with_demo.pdf)

- **Pgpool - What, Why, and Where?**
    - Speaker: Muhammad Usama
    - Slides: [PGConf.EU 23 Pgpool - What, Why, Where.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4808/slides/449/PGConf.EU%2023%20Pgpool%20-%20What,%20Why,%20Where.pdf)

- **Use Ansible to herd your Elephants!**
    - Speaker: Julian Markwort
    - Slides: [ansible_to_shepherd_your_elephants.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4942/slides/405/ansible_to_shepherd_your_elephants.pdf)

- **Don't Do High Availability, Do Right Availability**
    - Speaker: Greg Vernon
    - Slides: [Right_avail_pgconfeu.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4929/slides/413/Right_avail_pgconfeu.pdf)

- **Postgres vs. Linux filesystems**
    - Speaker: Tomas Vondra
    - Slides: [postgres-vs-filesystems-pgconfeu-2023.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4670/slides/436/postgres-vs-filesystems-pgconfeu-2023.pdf)

- **How we execute PostgreSQL major upgrades at GitLab, with zero downtime.**
    - Speaker: Alexander Sosna
    - Slides: [2023.pgconf.eu Zero Downtime PostgreSQL Upgrades.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4791/slides/439/2023.pgconf.eu%20Zero%20Downtime%20PostgreSQL%20Upgrades.pdf)

- **Postgres 16 highlight: Logical decoding on standby**
    - Speaker: Bertrand Drouvot
    - Slides: [pgconfeu2023_logical_decoding_on_standby.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4867/slides/433/pgconfeu2023_logical_decoding_on_standby.pdf)

- **PostgreSQL Replication: 20 Pitfalls and Solutions**
    - Speaker: Julian Markwort
    - Slides: [postgres_replication_pitfalls_and_solutions.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4937/slides/417/postgres_replication_pitfalls_and_solutions.pdf)

- **What can't pgBackRest do for you?**
    - Speaker: Stefan Fercot
    - Slides: [20231215_PGConfEU_What-cant-pgBackRest-do-for-you.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4698/slides/423/20231215_PGConfEU_What-cant-pgBackRest-do-for-you.pdf)

- **PostgreSQL Write-Ahead Logging (WAL): The Internals of Reliability and Recovery**
    - Speaker: Hamid Akhtar
    - Slides: [PostgreSQL Write-Ahead Logging (WAL)_ The Internals of Reliability and Recovery.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4896/slides/429/Hamid%20Akhtar%20-%20PostgreSQL%20Write-Ahead%20Logging%20(WAL)_%20The%20Internals%20of%20Reliability%20and%20Recovery.pdf)

- **Professional PostgreSQL monitoring made easy**
    - Speaker: Pavlo Golub
    - Slides: [Monitoring PostgreSQL made simple - pgconf.eu'23.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4696/slides/426/Monitoring%20PostgreSQL%20made%20simple%20-%20pgconf.eu'23.pdf)

- **Leveraging pgBadger for Effective PostgreSQL Troubleshooting**
    - Speaker: Alicja Kucharczyk
    - Slides: [Leveraging pgBadger for Effective PostgreSQL Troubleshooting-pgconfeu.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4720/slides/421/Leveraging%20pgBadger%20for%20Effective%20PostgreSQL%20Troubleshooting-pgconfeu.pdf)

### Developer

- **Vectors are the new JSON**
    - 向量是一个历经多个世纪研究的数学概念，但在数据库系统中，它们在高效存储和检索方面仍面临许多挑战。AI/ML的易用性提高导致人们对将向量数据与应用数据存储在一起产生了浓厚的兴趣，从而带来了一些独特的挑战。PostgreSQL 在 JSON 成为 Web 通用语言时曾经历过类似的情况。那么，使用 PostgreSQL 管理向量数据应该注意哪些挑战呢？
    - Speaker: Jonathan S. Katz
    - Slides: [pgconfeu2023_vectors.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4592/slides/435/pgconfeu2023_vectors.pdf)

- **A journey into postgresql logical replication**
    - Speaker: José Neves
    - Slides: [A journey into postgresql logical replication.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4773/slides/427/A%20journey%20into%20postgresql%20logical%20replication.pdf)

- **How I found my Pokémon cards thanks to Postgres: an AI journey**
    - Speaker: Matt Cornillon
    - Slides: [how-i-found-my-pokemon-cards-thanks-to-postgres-an-ai-journey](https://github.com/Matthieu68857/how-i-found-my-pokemon-cards-thanks-to-postgres-an-ai-journey)

- **Beginner's Guide to Partitioning vs. Sharding in Postgres**
    - Speaker: Claire Giordano
    - Slides: [Beginner's Guide to Partitioning vs. Sharding in Postgres | Claire Giordano | PGConf EU 2023](https://speakerdeck.com/clairegiordano/beginners-guide-to-partitioning-vs-sharding-in-postgres-claire-giordano-pgconf-eu-2023)

- **Counting things at the speed of light with roaring bitmaps**
    - Speaker: Ants Aasma
    - Slides: [roaring.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4762/slides/408/roaring.pdf)

- **It's Not You, It's Me: Breaking Up with Massive Tables via Partitioning**
    - Speaker: Chelsea Dole
    - Slides: [Breaking Up Massive Tables with Partitioning.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4699/slides/438/Breaking%20Up%20Massive%20Tables%20with%20Partitioning.pdf)

- **Learning advanced SQL the weird (and hard) way**
    - Speaker: Lætitia AVROT
    - Slides: [https://tinyurl.com/ye9pxyeu](https://tinyurl.com/ye9pxyeu)

- **Should I use JSON in PostgreSQL?**
    - Speaker: Boriss Mejias
    - Slides: [bmejias_should_I_use_json.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4900/slides/441/bmejias_should_I_use_json.pdf)

- **Blazingly Fast Message Queue on Postgres with Rust**
    - Speaker: Adam Hendel
    - Slides: [PGConfEU 2023 - MQ on PG.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4891/slides/422/PGConfEU%202023%20-%20MQ%20on%20PG.pdf)

- **PostGIS and pgRouting: Extensions for spatial data in PostgreSQL**
    - Speaker: Marion Baumgartner
    - Slides: [postGISPGRoutingPresentation.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4828/slides/440/postGISPGRoutingPresentation.pdf)

### Internals

- **Multi-threaded PostgreSQL?**
    - 将 PG 改为多线程架构似乎是一个很困难的事情，但比起 10 年前要可能性更大了，操作系统、编译器的支持更加完备，依赖的库也都提供 thread-safe 的接口。
    - Speaker: Heikki Linnakangas
    - Slides: [Multi-threaded PostgreSQL .pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4845/slides/407/Multi-threaded%20PostgreSQL%20.pdf)

- **Making your patch more committable**
    - Speaker: Melanie Plageman
    - Slides: [Making your patch more committable](https://speakerdeck.com/melanieplageman/making-your-patch-more-committable)

- **The path to using AIO in postgres**
    - Speaker: Andres Freund
    - Slides: [path-to-aio.pdf](https://anarazel.de/talks/2023-12-14-pgconf-eu-path-to-aio/path-to-aio.pdf)

- **PostgreSQL hacker tips**
    - Speaker: Michael Paquier
    - Slides: [PGConfEU2023_PostgreSQL_Hacker_Tips.pdf](https://www.postgresql.eu/events/pgconfeu2023/sessions/session/4859/slides/399/PGConfEU2023_PostgreSQL_Hacker_Tips.pdf)
