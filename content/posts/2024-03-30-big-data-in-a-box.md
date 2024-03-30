---
author: pg-x
title: "Big data in a box"
date: 2024-03-30T09:25:53+08:00
tags: []
ShowToc: false
TocOpen: false
---

> In memory of Simon Riggs.

PostgreSQL 的核心贡献者 Simon Riggs 于 2024 年 3 月 26 日驾驶私人飞机在杜克斯福德机场坠毁，并在事故中丧生。

Simon 负责过许多 PostgreSQL 的企业级特性，包括时间点恢复（PITR）、热备份（hot standby）和同步复制（synchronous replication）。同时他也是 2ndQuadrant 的创始人，该公司雇佣了许多 PostgreSQL 的开发人员，并后来成为 EDB 的一部分，Simon 在 EDB 担任 Postgres Fellow 直到退休。

为了让更多的人了解或记住 Simon Riggs，这里分享一个 Josh Berkus 讲述的关于 Simon Riggs 有趣的故事。

-- Start of the story --

2006~2008 期间，Josh Berkus 和 Simon Riggs 在 Greenplum 共事，一起合作开发 Greenplum 的资源管理系统。有一次，Simon Riggs 飞往加州的 San Rafael，在抵达的第一个晚上，合作伙伴们带他们去了一家冒充法式小酒馆的连锁餐厅，餐厅里有一个龙舌兰酒吧台，这是 Simon 第一次尝试龙舌兰（Josh 印象中 Simon 不太喜欢）。

Simon 之前在美国待过的时间不多，也从未去过连锁餐厅。这些美国连锁餐厅的份量通常都非常大；Simon 点的牛排配薯条可能有 11 盎司的牛肉和一个装满薯条的大篮子。Simon 几乎只吃了三分之一。

要理解接下来发生的事情，需要知道：**在英式英语中，`box` 这个词也是女性生殖器的俚语**，并且英国人通常不会把餐厅的剩菜打包带走。

当 Simon 明显不再能吃下去时，一位身材娇小、穿着迷人的女服务员走过来问他：**Do you want to put it in a box?** (服务员的意思是：你要把剩下的打包带走吗？)

**DO I WANT A WHAT?!?!!?** Simon 怒吼道（由于文化差异 Simon 理解成了 box 的俚语 🤪）。

服务员尖叫一声，把盘子扔到桌子上，逃离了现场。

Josh 向 Simon 解释了情况，他们向服务员道了歉（并给了很多小费）。

此后，**"box" 成为 Greenplum 的一个梗**；每当会议陷入困境时，总会有人说：**"maybe we should put it in a box"**。

Greenplum 最初的 SW/HW 产品被昵称为 **"Big data in a box"**。

-- End of the story --

向 Postgres 最杰出的开发者之一，也是一个善于发现幽默的人 Simon Riggs 致敬。

感谢 PostgreSQL contributor 何建给我分享这个故事的[链接](https://m6n.io/@fuzzychef/112172393647826741)。

### References

- [Remembering Simon Riggs](https://www.postgresql.org/about/news/remembering-simon-riggs-2830/) by PostgreSQL Core Team
- [Pilot killed in plane crash is named by friends](https://www.bbc.com/news/articles/cjex992z0wlo)
