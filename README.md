# MSSQL
数据库相关问题学习补充

## 文章导航

[包含列的索引（mssql-index-include）](src/docs/mssql-index-include.md)

# MYSQL

## 在设计主键的时候应该注意什么？

首先要根据业务场景对数据量进行估量，因为在设计主键字段类型时，如果核心表的数据量是非常大的，那么**自增长主键**应该设置为 `bigint` 类型。因为 `int` 类型的范围只有 `-2147483648 - 21474836487`，对于日志等监控信息表来说，每天的数据量都是非常惊人的，所以很快就会到达 int 的边界。当然，有很多人选择定时清空数据，但是要知道，当主键设置为自增长，总的主键值是不会跟着清空的，除非也要定时重置 auto_crement。

在分布式架构中，更推荐用**字符串做主键**

## 在设计金额字段的时候是不是一律用 `decimal`？

我们一般习惯性的将金额字段设置为 `decimal`，因为可以设置准确的设置精度。但是最佳实践是将这个字段设置为**整型字段**。也就是说我们的金额单位是 “分”。如 1 元，我们存储 100。

这种选择有两方面的优势：

- 存储：因为 `decimal` 是变长字段，这样随着值的大小，decimal 所占的字节也会有变化。这样就会导致数据库存储空间有空洞，导致碎片问题。而整型是个定长字段，所以对内存友好， 不会导致内存碎片问题。
- 性能：在海量并发的场景下，定长的性能远远好于变长字段（为什么？论据）

在 MYSQL 8.0 之前还有一个字增长的问题，那就是**字增长不会持久化**，自增值可能会存在回溯问题：在表中有三条记录，我们删除三条表，由于字增长特性，我们再添加一条数据，次数的主键值为 4。但是如果在删除后添加前，mysql 发生了故障重启了，那么这个时候我们新增一条记录主键值是 3。

## MySQL 参数配置查询

flush 会导致刷脏页，即将 MySQL 会把内存的数据同步写入到磁盘中，这个过程就称为 flush，这是很耗内存的。

脏页比例是通过 `Innodb_buffer_pool_pages_dirty/Innodb_buffer_pool_pages_dirty` 计算出来的

```mysql
select VARIABLE_VALUE into @a from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_dirty';
select VARIABLE_VALUE into @b from global_status where VARIABLE_NAME = 'Innodb_buffer_pool_pages_total';
select @a/@b;
```

显示变量状态参数

```mysql
mysql global status;
```

查看 SQL 执行进程的情况，可以查看当前语句执行处于什么状态：

```mysql
show processlist;  -- 显示进程相关的信息，id，当前状态 state, info 等
```

也可以通过查询具体的 process_id 的信息：

```mysql
mysql> select * from information_schema.processlist where id = 1;
```

通过查询 `sys.schema_table_lock_waits` 就可以查出造成阻塞的 process_id，找出来 kill 掉

```mysql
mysql> select blocking_pid from sys.schema_table_lock_waits;
```

MySQL 5.7 版本可以通过 `sys.innodb_lock_waits` 查询出谁在占用锁：

```mysql
mysql> select * from sys.innodb_lock_waits where locked_table = `'test'.'t'`;
```



## Count 不同用法的性能差异

```mysql
select count(*) from table;
select count(主键id) from table;
select count(1) from table;
select count(列字段) from table;
```

count() 是一个聚合函数，**对于返回的结果集要一行行地判断，如果 count 函数的参数不是 null，累加值+1，否则不加。最后返回这个累加值。**

- select(主键id)：会遍历整张表，每一行的主键都取出来，返回 server 层。server 层拿到 id 后，判断不为 null 的累加值 +1
- count(1): 遍历整张表，但不取值。server 层直接返回每一行，直接累加值 +1
- count(字段)：遍历整张表，取对应的字段，返回 server 层。server 判断该值是否 null，决定累加值 +1
- count(*): MySQL 内部做了优化，不取值，直接按行取值累计值 +1

性能比较：

count(*) 约等于 count(1) > count(主键id) > count(字段)

## UUID 与 自增长 ID 的选择

其实我们在做分布式应用时，我们在选择主键的类型选择时，绝大多数都会在 UUID 和分布式 Id（如 facebook 的雪花 Id） 中选择。UUID 生成规则其实与雪花Id这种分布式Id类似。UUID 的生成规则如下

```
UUID = 时间低（4字节）- 时间中高+版本（4字节）- 始终序列 - MAC地址
```

它的生成规则就注定它天生就是带有分布式性质的。前 8 个字节中的 60 位用于存储时间，4 位存储 UUID 的版本号，这个时间是从 1582-10-15 00:00:00 到现在的 100 ns 的计数。

而且要注意，存储时间位时，时逆序的。时间低的放在前面。所以并不是单调递增的。**非随机值在插入时会产生离散 I/O**，从而产生性能瓶颈。这是人们对于选择 UUID 做为主键最大的顾虑。

而 MySQL 为了解决这个问题，8.0 推出了函数 `UUID_TO_BIN`，它可以对 UUID 字符串进行以下处理：

- **通过参数讲时间高位放在最前面，解决了 UUID 插入时序乱问题。**
- 去掉无用的字符串 "-"，精简存储空间。
- **将字符串转换期二进制存储，空间最终从之前的 36 个字节缩短成 16 个字节。**

那么在 MySQL8.0 之前该怎么解决呢？

```mysql
CREATE FUNCTION MY_UUID_TO_BIN(_uuid BINARY(36))
    RETURNS BINARY(16)
    LANGUAGE SQL  DETERMINISTIC  CONTAINS SQL  SQL SECURITY INVOKER
    RETURN
        UNHEX(CONCAT(
            SUBSTR(_uuid, 15, 4),
            SUBSTR(_uuid, 10, 4),
            SUBSTR(_uuid,  1, 8),
            SUBSTR(_uuid, 20, 4),
            SUBSTR(_uuid, 25) ));

CREATE FUNCTION MY_BIN_TO_UUID(_bin BINARY(16))
    RETURNS  CHAR(36)
    LANGUAGE SQL  DETERMINISTIC  CONTAINS SQL  SQL SECURITY INVOKER
    RETURN
		LCASE(CONCAT_WS('-',
		HEX(SUBSTR(_bin,  5, 4)),
		HEX(SUBSTR(_bin,  3, 2)),
		HEX(SUBSTR(_bin,  1, 2)),
		HEX(SUBSTR(_bin,  9, 2)),
		HEX(SUBSTR(_bin, 11)) ));

```

UUID 缺点：

- UUID 可读性差，不带有业务属性
- 分布式数据库下，可能会存在重复的 UUID（联想生成规则）

在选择时我们还可以选择第三方生成机制，如 redis 自增以及 zookeeper 拨号机制来生成主键（分段+二级/三级缓存提升并发性能），又如 mongodb 的 objectid 也是 12 字节全局唯一的。

