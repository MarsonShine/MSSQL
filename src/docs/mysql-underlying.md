# MySQL 底层原理

在执行 SQL 语句的时候，如执行 "SELECT * FROM User"，然后 MySQL 返回给我们结果集合。这里面看似简单，实际上经过了 5 个步骤：1. 建立链接；2. 查询缓存（MySQL8.0 之后删除了缓存机制）3. 词法分析；4.优化器；5. 执行器

## 建立连接

首先是根据 MySQL 服务器的账号和密码来与 MySQL 建立连接获取查询权限：`MYSQL -uroot -p123456`。一般是选择长连接方式。因为建立连接的开销是比较大的，短链接会在一个段时间内处理空闲状态的下被回收掉（长连接的 TTL 的时间默认是 8 小时）。但是长连接也是有一定的损耗的，每个长连接都需要消耗内存的。当连接的端口比较多时，内存占用就会越来越多，因为这些资源只有在连接断开后才释放，所以最后就会导致 MySQL 服务器崩溃而重启。

而解决方案就是：

- 定期断开长连接。使用一段时间，或者程序里面判断执行过一个占用内存的大查询后，断开连接，之后要查询再重连。
- 如果是 MySQL5.7 及以上版本，可以在每次执行一个比较大的操作后，通过执行 `mysql_reset_connection` 来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态

## 查询缓存

这个我建议是不要使用，因为建立缓存也是需要占用内存的。虽然在有些时候会加快查询效率，但是一旦发生信息的改写，那么就会导致缓存失效。缓存有些场景是好用的，特别是一些相对 “稳定” 的数据，比如数据统计等。MySQL 就支持针对特定的语句进行缓存：

```mysql
mysql> select SQL_CACHE * from T where ID = 10;
```

⚠️：MySQL8.0 及以上版本已经把查询缓存删了。

## 词法分析

如果没有命中缓存，就开始下一个阶段，词法分析。这个阶段 MySQL 会分析要执行的 SQL 是否正确。比如识别 select、from、where 等关键字，把 T 识别成表名，以及 ID 识别成 T 表中的列明。如果不符合规则，就会提示语法错误。

## 优化器

词法分析过了之后，就会有 MySQL 内部的优化器来选择优化项，如决定使用什么索引，使用哪个索引，联表的连接顺序。最后再交给执行器执行。如：

```mysql
mysql> select * from t1 join t2 using(ID) where t1.c = 10 and c2.d = 20;
```

这段语句在优化器就会有两种优化顺序：

- 选取 t1.c = 10 的记录，然后再根据 ID 关联 t2 表，然后再判断 t2 里 d 的值是否是 20
- 选取 t2.d = 10 的记录，然后再根据 ID 关联 t1 表，然后再判断 t1 里 c 的值是否是 10

内部优化器自己会根据优劣选择顺序。

## 执行器

- InnoDB 引擎接口取这个表的第一行，判断是否符合查询条件，如果不是就会跳过，是就会存到结果集中；
- 继续取下一行，重复相同的判断，直到最后一行；
- 将结果集作为结果返回客户端；

## 数据恢复

MySQL 内部采用 [WAL（预写日志）](https://github.com/MarsonShine/MS.Microservice/blob/master/docs/patterns-of-distributed-systems/Write-Ahead-Log.md) 做到数据恢复的，依靠的是 redo log 和 bin log。redo log 是 InnoDB 引擎特有的物理日志；bin log 是逻辑日志，属于 MySQL 默认引擎内置的日志。**redo log 记录的是 “在某个数据页上做了什么修改”；bin log 记录的是 SQL 语句的原始逻辑，比如 “给 ID=2 的一行数据的列 C 加 1”。**

![](asserts/2e5bff4910ec189fe1ee6e2ecc7b4bbe.png)

​																				此图出自即刻时间《MySQL 45讲实战》