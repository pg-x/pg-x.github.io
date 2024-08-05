---
author: pg-x
title: "Postgresql Hacking Workshop 中文内核小组"
date: 2024-08-05T12:32:39+08:00
tags: []
ShowToc: false
TocOpen: false
---

PostgreSQL 作为世界上最先进的开源关系型数据库之一，拥有一个活跃的全球开发者社区。为了进一步培养更多的开发者，Robert Haas 发起了 [PostgreSQL Mentoring Program](/posts/2024-08-03-postgresql-mentoring-program/)。

PostgreSQL Hacking Workshop 作为其中的一个项目，旨在促进参与者之间就相关的视频内容，提出问题、分享见解并进行的深入讨论和思想交流。

本着同样的宗旨，我创建了 PostgreSQL Hacking Workshop 中文内核小组，期望创建一个充满热情的中文 PG 内核开发者社区。

该小组的重点包括：

- 为 PG 内核开发者中文用户提供一个学习、讨论和协作 PostgreSQL 开发的平台
- 组织文化和语言上适合中文使用者的研讨会和在线讨论
- 通过打破语言障碍并提供必要的支持，鼓励参与全球 PostgreSQL 社区
- 记录和分享有价值的讨论和发现，以帮助未来的内核开发人员和贡献者

该小组采取的方式与 PostgreSQL Hacking Workshop 相同，除了学习 PostgreSQL Hacking Workshop 筛选的视频之外，中文内核小组每个月还会挑选一个 PostgreSQL 的成熟特性（包括但不限于博客文章、书籍中的一章）和一个正在研发中的特性（包括但不限于当前研发版本中已经合并或正在讨论的特性）以及一个成熟的 PostgreSQL 生态的 extension。

我们通过微信群进行讨论，每个月的主题都会在当月的第一个周末发布到相关的微信群中（同时以 [Github issue](https://github.com/pghacking/workshop-cn/issues) 的方式进行跟踪）。在这个月内，群成员可以就这些话题进行深入讨论，并鼓励有能力的群成员到上游社区发表自己的观点。

我们期望以这种方式促进中文内核小组开发成员之间的交流和学习，提升中文开发者在 PostgreSQL 全球社区中的参与度和影响力。通过定期的讨论和分享，我们可以共同推动 PostgreSQL 技术的发展和应用。

[第一期 Workshop](https://github.com/pghacking/workshop-cn/issues/1) 讨论的话题:

- **视频**: https://www.youtube.com/watch?v=XA3SBgcZwtE
- **书籍**: [PostgreSQL 14 internals](https://edu.postgrespro.com/postgresql_internals-14_en.pdf), ch1 Introduction
- **邮件列表**: [why there is not VACUUM FULL CONCURRENTLY?](https://www.postgresql.org/message-id/flat/CAFj8pRDK89FtY_yyGw7-MW-zTaHOCY4m6qfLRittdoPocz%2BdMQ%40mail.gmail.com)
- **插件**: [pg_squeeze](https://github.com/cybertec-postgresql/pg_squeeze)

**why there is not VACUUM FULL CONCURRENTLY?** 这个邮件列表讨论的 idea 是将 pg_squeeze 的能力集成到内核中，减少 VACUUM FULL 锁表的时间，所以我选择了 pg_squeeze 作为本月的另一个主题。

在昨天的文章发布微信公众号的二维码之后，已经有 70 余人扫码入群，但讨厌的是，有些人进来就发广告，所以我把扫码入群的功能关闭了，后续如果想加入群聊，需要添加我的个人微信 zhjwpku，我会问一些 PG 相关的知识以确保群成员都是 PG 相关的。
