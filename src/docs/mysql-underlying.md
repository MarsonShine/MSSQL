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

### Redo log

MySQL 如果每次操作都需要写磁盘，然后磁盘也要转动磁道找到对应的记录执行更新，整个过程会造成非常大的 I/O 消耗和查询成本。**MySQL 通过缓存一批操作到日志文件，然后在一次性写到磁盘来提升效率，这就是 WAL 技术。**

而 InnoDB 引擎的 redo log 就会先把记录写到 redo log 里面，并更新内存，这个时候操作就结束了。InnoDB 会在适当的时候把这些操作记录写入到磁盘里面。

redo log 是固定大小的，比如可以配置为一组 4 个文件，每个文件的大小是 1GB。**数据结构是一个环形结构。当写入的数据达到最大值时就会从头开始循环写。** 

有了 Redo log，InnoDB 就能保证了 MySQL 在发生崩溃重启，之前的提交的记录都不会丢失，从而提供了 crash-safe 的能力。

具体执行过程是：

在一个事务开启并执行相关操作的过程中，会先写入 redo log buffer 中。然后待事务提交时（commit）就会将 redo log buffer 的内容写入（wirte）到磁盘并根据同步策略将其持久化到磁盘上。

> redo log buffer，在写入 redo log file 之前会先写入一个缓冲区，来提供效率。
>
> 事务提交时才会将 buffer 上的操作写入（但未持久化）到磁盘上，这句话的意思是将数据写到文件系统的 page cache 上。**也就是说当 MySQL 实力因故障而重启时数据是不会丢失的，但是如果主机发生故障而重启，那么 page cache 上的事务操作就会丢失**。

同步策略 `innodb_flush_log_at_trx_commit` 具体设置为：

- 0，每次事务提交都将 redo log 留在 redo log buffer 中（这样 MySQL 重启就会丢失这些事务操作）
- 1，每次事务提交都将 redo log 持久化到磁盘
- 2，每次事务提交都将 redo log 写到 page cache 

关于 redo log 写入的流程可以看下面的例子：

```
begin;
insert into t1 ...
insert into t2 ...
commit;
```

在一个事务的更新过程，日志是要写多次的，如上面的事务操作。这个事务要往两个表插入数据，在插入的过程中，生成的日志都要先保存下来（WAL），但又不能在还没 commit 的时候直接写到 redo log 文件里。

所以 redo log buffer 就是一个内存块，用来先存 redo 日志。也就是说，在执行第一个 insert 的时候，数据的内存被修改了，redo log buffer 也写入了日志。但是，真正把日志写入到 redo log 文件，是在执行 commit 语句的时候执行的。

### bin log

除了 redo log 的 crash-safe 能力之外，还要有 MySQL 自带的 bin log 的能力。它记录每个事务操作的原始逻辑，**它是追加写的，是无法被覆盖的** 。

同样的，为了减少 I/O 的成本，bin log 也有 redo log buffer 类似的作用—— bin_log cache。**当一个事务操作，某些 SQL 操作将数据更新到了内存中，此时数据页就会发生变化，即就会把这个操作记录到 redo log 中，此时处于 prepare 状态，目的是为了告诉执行器执行完成了，可以随时提交事务。那么这个时候紧接着就会将这个操作的 bin log 写入到 buffer 中，并根据同步策略持久化到磁盘中。最后提交事务。（即先写入 redo log，后写入 bin log）**

那么具体的执行过程：

当事务开始时，日志记录到 bin_log cache 中。

当事务结束时，bin_log cache 会写入到文件系统的 page cache 中（同样，次事务操作没有持久化到磁盘），然后清空 bin_log cache，并根据 fsync 同步策略将 page cache 持久化到磁盘。

同步策略 `sync_binlog` 具体设置为：

- 0，每次事务提交都会执行 write（即写入 page cache）
- 1，每次事务提交都会执行 fsync（即持久化磁盘）
- N，每次提交事务都会执行 write，积累到 N 个事务后执行 fsync

其实由这两种日志的同步策略就能看到，如果设置 “双1” 配置，就能确保数据是可以恢复的。

### 为什么需要 redo log + bin log 才能保证数据能正确恢复

先了解崩溃恢复的判断规则：

1. 如果 redo log 里面的事务是完整的，即 redo log 里面含有 commit 标识，则直接提交；
2. 如果 redo log 里面的事务只有 prepare，则就进而判断 binlog 是否存在并是完整的：
   1. 如果是，则提交事务
   2. 否则，回滚事务

那么 MySQL 是如何知道 binlog 是完整的呢？

其实一个事务的 binlog 本身就是有一个完整的格式的：

- statement 格式的 binlog，最后都会有 COMMIT；
- row 格式的 binlog，最后都会有一个 XID event。

此外还有一个 binlog-checksum 来验证 binlog 的内容是否正确。

那么 redo log 和 bin log 如何结合起来呢？

两个独立的文件要关联起来，势必会加入一个共同体，**即 XID，这是一个共同的字段。**

在崩溃恢复的时候，会按顺序扫描 redo log：

- 如果碰到既有 prepare 又有 commit 的 redo log，则直接提交
- 如果只有 prepare，则根据 XID 去扫描 binlog 找到对应的事务回放事务操作。

## 数据复制怎么保证主备一致

// TODO