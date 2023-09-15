---
author: pg-x
title: "PostgreSQL 返回列存格式"
date: 2023-09-15T16:34:16+08:00
tags: [arrow]
ShowToc: false
TocOpen: false
---

PostgreSQL 是一个关系型数据库，无论通过 libpq 还是 jdbc 去进行查询，其返回的数据格式都以行（Row/Record）为单位。不过现在出现了一种新的可能，2023/09/13，Sutou Kouhei 发布了 [Apache Arrow Flight SQL adapter for PostgreSQL](https://github.com/apache/arrow-flight-sql-postgresql) [0.1.0 版本](https://arrow.apache.org/blog/2023/09/13/flight-sql-postgresql-0.1.0-release/)，这意味着今后我们可以用 [arrow flight](https://arrow.apache.org/docs/format/Flight.html) 来访问保存在 PostgreSQL 中的数据，而且返回的是 Arrow 定义的列存格式！

### 编译安装

```shell
git clone https://github.com/apache/arrow-flight-sql-postgresql.git
cd arrow-flight-sql-postgresql
mkdir build && cd build
meson setup -Dexample=true ..
ninja
sudo meson install
```

上面操作结束后会把生成的 arrow_flight_sql.so 拷贝到 `pg_config --libdir` 指定的目录，然后修改 postgresql.conf 加上下面的语句并重启实例。

```shell
shared_preload_libraries = 'arrow_flight_sql'
```

### 功能测试

该项目提供了几个客户端程序样例，上面的编译选项 `-Dexample=true` 把样例程序也编译了，核心代码逻辑如下，这里执行的 SQL 语句被我替换成了带 join 的查询:

```C++
// Start query
arrow::Status
run()
{
	arrow::flight::FlightCallOptions call_options;
	ARROW_ASSIGN_OR_RAISE(auto sql_client, connect(call_options));
	ARROW_ASSIGN_OR_RAISE(
		auto info,
		sql_client->Execute(call_options, "select * from t1, t2;"));
	for (const auto& endpoint : info->endpoints())
	{
		ARROW_ASSIGN_OR_RAISE(auto reader,
		                      sql_client->DoGet(call_options, endpoint.ticket));
		while (true)
		{
			ARROW_ASSIGN_OR_RAISE(auto chunk, reader->Next());
			if (!chunk.data)
			{
				break;
			}
			std::cout << chunk.data->ToString() << std::endl;
		}
	}
	return sql_client->Close();
}
// End query
```

connect 函数调用 ` arrow::flight::FlightClient::Connect` 去连接 arrow_flight_sql 暴露的 endpoint，执行 sql 语句并将返回结果输出。上面的查询如果用 psql 的到的输出如下:

```SQL
postgres=# table t1;
 a |   b
---+-------
 1 | hello
 2 | hello
 3 | hello
 4 | hello
 5 | hello
(5 rows)

postgres=# table t2;
 b
---
 1
 2
 3
 4
 5
(5 rows)

postgres=# select * from t1, t2;
 a |   b   | b
---+-------+---
 1 | hello | 1
 2 | hello | 1
 3 | hello | 1
 4 | hello | 1
 5 | hello | 1
 1 | hello | 2
 2 | hello | 2
 3 | hello | 2
 4 | hello | 2
 5 | hello | 2
 1 | hello | 3
 2 | hello | 3
 3 | hello | 3
 4 | hello | 3
 5 | hello | 3
 1 | hello | 4
 2 | hello | 4
 3 | hello | 4
 4 | hello | 4
 5 | hello | 4
 1 | hello | 5
 2 | hello | 5
 3 | hello | 5
 4 | hello | 5
 5 | hello | 5
(25 rows)
```

Flight 样例程序输出如下，每一列单独输出，即返回的结果是列存格式！🥳

```shell
vagrant@ubuntu-focal:/vagrant/arrow-flight-sql-postgresql/build$ ./example/flight-sql/query-ad-hoc
a:   [
    1,
    2,
    3,
    4,
    5,
    1,
    2,
    3,
    4,
    5,
    ...
    1,
    2,
    3,
    4,
    5,
    1,
    2,
    3,
    4,
    5
  ]
b:   [
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    ...
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello",
    "hello"
  ]
b:   [
    1,
    1,
    1,
    1,
    1,
    2,
    2,
    2,
    2,
    2,
    ...
    4,
    4,
    4,
    4,
    4,
    5,
    5,
    5,
    5,
    5
  ]
```

除 SELECT 之外，该扩展还支持 INSERT、UPDATE、DELETE 及 prepare statement。

### 实现原理

Apache Arrow Flight SQL adapter for PostgreSQL 服务端的代码都在 src/afs.cc 文件中，_PG_init 中首先启动一个名为 **arrow-flight-sql: main** 的 bgworker，main 又启动一个名为 **arrow-flight-sql: server**，server 会启动 FlightSQLServer，FlightSQLServer 继承了 `arrow::flight::sql::FlightSqlServerBase`，在各个接口的 override 函数中将 Flight SQL 的各种查询分发给 Proxy，Proxy 在 Flight 客户请求到来的时候创建一个 session 放到共享内存 hash 表里，然后通知 main 去处理连接，为每一个连接创建一个名为 **arrow-flight-sql: executor: <session_id>** 的 bgworker。executor 则使用 SPI(Server Programming Interface) 接口与数据库进行交互并将结果转为 Arrow Format（如需返回结果，比如 select）。大致的架构图如下所示:

![architect](/images/apache_flight_adapter.png)


### 小结

Apache Arrow 格式是专为带类型的表数据快速交换而设计的。如果你想通过 SELECT 查询或通过 INSERT/UPDATE 更新大量数据，使用 Apache Arrow Flight SQL 比使用 PostgreSQL 原本的传输协议更高效。相信这会是一个非常激动人心的新特性。 🎉🎊
