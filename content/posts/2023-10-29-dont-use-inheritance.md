---
author: pg-x
title: "Don't Use Table Inheritance"
date: 2023-10-29T18:07:07+08:00
tags: []
draft: false
ShowToc: false
TocOpen: false
---

PostgreSQL 是一个对象关系数据库管理系统（[object-relational database management system](https://www.postgresql.org/docs/current/intro-whatis.html))，如果你看维基百科的定义，[ORDBMS](https://en.wikipedia.org/wiki/Object%E2%80%93relational_database) 是一种拥有面向对象特性的关系型数据库，对象、类、继承（Inheritance）等概念直接在数据库 schema 和查询语言层面支持。

> An object–relational database (ORD), or object–relational database management system (ORDBMS), is a database management system (DBMS) similar to a relational database, but with an object-oriented database model: objects, classes and inheritance are directly supported in database schemas and in the query language.

一定会有人认为 **Inheritance** 是 PostgreSQL 被称为 ORDBMS 的主要原因，比如 David Johnston(PG contributor) 在 slack 上的回复:

![ORDBMS](/images/postgres_ordbms.png)

但 PostgreSQL 社区维护的 [Don't Do This](https://wiki.postgresql.org/wiki/Don%27t_Do_This) 里提示了 **[Don't use table inheritance](https://wiki.postgresql.org/wiki/Don%27t_Do_This#Don.27t_use_table_inheritance)**，所以 Inheritance 一定不是 PostgreSQL 称作 ORDBMS 的唯一原因。ChatGPT 给出的答案我认为具有一定参考价值:

![PostgreSQL Object Meaning](/images/postgresql_object_meaning.png)

### 为什么不建议用 Table Inheritance

我们来看一个 PostgreSQL 邮件列表中出现过的例子:

```SQL
CREATE TABLE PERSON (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    dob DATE
);

CREATE TABLE CUSTOMER (
    registration_date DATE NOT NULL,
    contact VARCHAR(255)
) INHERITS (person);

INSERT INTO PERSON VALUES (1, 'Fulano', '1965-06-07');
INSERT INTO CUSTOMER VALUES (2, 'Beltrano', '1980-10-07', '2023-10-10', '5561999999999');
```

创建一个父表 `PERSON`，id 为主键，子表 `CUSTOMER` 继承 `PERSON`，两张表各插入一条记录。如果现在我们向 `CUSTOMER` 插入一条数据，能成功吗？

```SQL
INSERT INTO CUSTOMER (id, name, dob, registration_date, contact)
SELECT id, name, dob, '2023-10-17', 'contact@example.com'
FROM person
WHERE id = 1;
```

答案是可以。查询父表得出的结果:

```SQL
postgres=# select * from person;
 id |   name   |    dob
----+----------+------------
  1 | Fulano   | 1965-06-07
  2 | Beltrano | 1980-10-07
  1 | Fulano   | 1965-06-07
(3 rows)
```

这难道不是主键冲突了吗？

父表和子表都存储了数据，父表只检查自身的约束（Constraints），子表的插入不会去检测父表上定义的约束。对父表的查询会把父表和子表的数据合并输出，从其查询计划可以看出代价估算也不准确。

```SQL
postgres=# explain select * from person;
                                QUERY PLAN
---------------------------------------------------------------------------
 Append  (cost=0.00..12.26 rows=84 width=524)
   ->  Seq Scan on person person_1  (cost=0.00..1.14 rows=14 width=524)
   ->  Seq Scan on customer person_2  (cost=0.00..10.70 rows=70 width=524)
(3 rows)
```

以上可能就是不建议使用 Table Inheritance 的主要原因，在分区表特性逐渐完善之后，Table Inheritance 特性在未来极有可能被[删除](https://www.postgresql.org/message-id/CAHyXU0y85n2ZzCPzZbxLk+WDeUjQXzoi=f1TumoYpPqKTWJVAw@mail.gmail.com):

![inheritance probably would have been removed](/images/postgres_table_inheritance_merlin.png)

如果你在考虑使用 [Table Inheritance](https://www.postgresql.org/docs/16/ddl-inherit.html)，请先评估一下 [Table Partitioning](https://www.postgresql.org/docs/current/ddl-partitioning.html) 是否能解决你的问题，可能会减少一些不必要的麻烦。
