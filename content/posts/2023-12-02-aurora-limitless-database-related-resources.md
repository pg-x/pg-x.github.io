---
author: pg-x
title: "Aurora Limitless Database Related Resources"
date: 2023-12-02T21:52:00+08:00
tags: [aurora]
ShowToc: false
TocOpen: false
---

AWS re:Invent 2023 发布了 Aurora Limitless Database，看起来类似 Citus，但其底层依赖 AWS 长久依赖的技术积淀，比如分布式事务使用了类似 Spanner True Time 的 EC2 TimeSync Service，又如利用 Caspian 来动态解决不同 shard 间的数据偏移。但其技术细节不对外公布，这里收集一些公开资料来学习其内部架构。AWS 真的是把 Serverless 做到了极致 👍🏻。

1. [AWS re:Invent 2023 - Monday Night Live Keynote with Peter DeSantis](https://www.youtube.com/watch?v=pJG6nmR7XxI)
2. [AWS re:Invent 2023 - Achieving scale with Amazon Aurora Limitless Database](https://www.youtube.com/watch?v=a9FfjuVJ9d8)
3. [Join the preview of Amazon Aurora Limitless Database](https://aws.amazon.com/blogs/aws/join-the-preview-amazon-aurora-limitless-database/)
4. [Hacker News](https://news.ycombinator.com/item?id=38447238)
