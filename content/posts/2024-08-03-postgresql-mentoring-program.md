---
author: pg-x
title: "PostgreSQL 导师计划"
date: 2024-08-03T20:54:25+08:00
tags: []
ShowToc: false
TocOpen: false
---

**[PostgreSQL Mentoring Program](https://www.postgresql.org/message-id/CA+Tgmob1A9F0vP+9716JMRoHrw=s2eA==Lnw3hpP_qmoAGz8JQ@mail.gmail.com)** 是由 PostgreSQL 社区核心开发人员同时也是 EDB 首席数据科学家的 [Robert Haas](https://www.linkedin.com/in/robertmhaas/) 发起的一项培养代码贡献者的计划。

在第一期的项目里，有 9 位 PostgreSQL Committer 自愿担任导师，对提交申请的代码贡献者进行一对一的指导。由于有 5 位 Committer 愿意同时指导 2 位同学，最终有 14 位申请者被选取。

在发起这项计划之前，Robert 在 PGConf.dev 2024 联合 Amit Langote, Masahiko Sawada, Melanie Plageman 进行了一个题为 Making PostgreSQL Hacking More Inclusive 的 Talk，社区希望越来越多的人参与到 PG 的代码开发当中。

{{< youtube ySg2aKYM0Bo >}}

这项计划对 Mentor 和 Mentee 的基本期望是二人至少每个月进行一次至少一小时的语音通话，在通话之前，Mentee 应该让 Mentor 知道他想谈论什么，Mentor 也会做一些相应的准备。在通话期间，Mentor 会尝试给 Mentee 一些有用的建议。Mentor 也会做一些其它的事情，比如 review Mentee 提交的 Patch，并私下进行沟通。两人中任何一方都可以随时出于任何原因或无原因结束导师关系，这时应该告诉该项目的组织者 Robert，他会进行进一步的安排。

除此之外，Robert 还创建了一个 Discord server，这个频道除了导师计划相关的人员，也有很多其他使用或开发 PG 的人员（后面有邀请链接），在这里除了可以讨论关于 PG 的任何话题之外，还有一个重要的事项是每月进行一次 [PostgreSQL Hacking Workshop](https://rhaas.blogspot.com/2024/07/mentoring-program-updates.html)，当前的形式是相关人员提前分享一些关于 PG 内核的 Talk，然后组织线上会议进行深入讨论，每次会议都会有一到两个 committer 参加，考虑到有些人可能害怕自己的问题比较 silly 而不敢开口，会议不进行录制。

每月的 Hacking Workshop 都需要进行申请，最终是否被邀请参会也根据时区、参会人数、申请表中提交的问题等因素进行综合考量，第一期 Workshop 观看的视频是 Robert Haas 在 CMU Database Group 分享的 PostgreSQL Optimizer Methodology，并在 8 月 8 日 进行第一次线上会议。

{{< youtube XA3SBgcZwtE >}}

### PostgreSQL 交流的平台

PostgreSQL 除了使用 mailing-list 进行讨论之外，也有一些官方或半官方的交流频道，除了上面讲的 Robert 创建的 PostgreSQL Hacker Mentoring 之外，还有一些其它的频道，我把对应的邀请列在下面:

- Postgres Slack: postgresteam.slack.com
- PostgreSQL Hacker Mentoring: https://discord.gg/bx2G9KWyrY
- People, Postgres, Data: https://discord.com/invite/bW2hsax8We

在上面三个群里提问大概率能得到有效的回答，PostgreSQL 也有自己的 [IRC 频道](https://www.postgresql.org/about/news/migration-of-postgresql-irc-channels-2216/)，但我的感觉是现在大部分 hacker 都在使用 Slack 或 Discord，IRC 上并没有太多的讨论。

另外，PG 生态的一些其它产品也有自己的交流频道，列举几个我平时关注的:

- Citus: citus-public.slack.com
- TimescaleDB: timescaledb.slack.com
- yugabyte-db: yugabyte-db.slack.com
- ParadeDB: paradedbcommunity.slack.com
- Cloudberry Database: cloudberrydb.slack.com

### 作为一名 Mentee 能学到什么

说回 PostgreSQL 导师计划，作为其中的一名 Mentee，我能学到什么呢？

1. 社区给我 match 的 Mentor 是 Amit Langote，他是在微软日本就职的一个印度人，虽然是印度人，但他没有任何印度口音（参见上面第一个视频），所以我每个月可以练习一个小时的英语口语
2. PostgreSQL mailing list 每条有上百条（可能不止）邮件，有导师的好处是能聚焦到某一个模块，而非在邮件列表里漫无目的的寻找感兴趣的话题，有时候取舍是件很难的事情
3. 能够和导师私下邮件沟通，而不必担心自己在邮件列表上问傻问题（当然 PG 有很好的包容性，问傻问题有时也会得到像 Tom Lane 这样的大佬的解答）

### Summary

PostgreSQL 拥有一个活跃而友好的社区，这个社区致力于推动 PostgreSQL 的良性发展，并且在不断地培养新的贡献者，使他们能够参与到 PostgreSQL 内核的开发中。希望更多的中国开发者能够参与到 PostgreSQL 的内核开发中。

最后，我创建了一个微信群 PostgreSQL Hacking，希望能在这个群里跟大家一起学习 PostgreSQL 内核知识。如果这个链接失效了可以加我个人微信 zhjwpku，我邀请进群；也可以点击**阅读原文**查看最新的二维码信息，我会定期更新。

![](/images/2024/IMG_8641.JPG)
