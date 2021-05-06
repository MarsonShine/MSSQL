# MySQL 索引

MySQL 索引采用的数据结构是 B+ 树。为什么会采用 B+ 树？

首先 B+ 树的查询性能好，是 O(log(N))，有人说哈希查询不是更高么。没错，哈希的确查询是要比 B+ 树要好得多，但是哈希是无序的，它不支持区间查询。

平衡二叉树的查询性能也很高，但是 MySQL 为什么也不用呢？原来就是这个 “二” 上，因为一个分支只有 2 个数据，当数据量非常大的时候，数据存在内存是不够的。必定会存到磁盘上，所以在访问磁盘的性能是非常低的。所以 MySQL 决定使用 ”N 叉树“。以 InnoDB 为例，这个 N 就是 1200。这棵树高是4的时候，就可以存1200的3次方个值，这已经17亿了。考虑到**树根的数据块总是在内存中的**，一个10亿行的表上一个整数字段的索引，**查找一个值最多只需要访问3次磁盘**。其实，树的第二层也有很大概率在内存中，**那么访问磁盘的平均次数就更少了**。

## MySQL 索引模型

我们在设置主键和索引时，其数据模型是不一样的。如我们创建一个表：

```mysql
mysql> create table T(
id int primary key, 
k int not null, 
name varchar(16),
index (k))engine=InnoDB;
```

其中 id 是主键（聚簇索引），k 是索引（非聚簇索引，也叫二级索引）。那么它们的数据模型分别为

![](asserts/dcda101051f28502bd5c4402b292e38d.png)

由上图可知：

- 主键索引的叶子节点存的是整行数据；
- 非主键索引的叶子节点内容是主键的值。

所以这在查询上是由区别的：

- 如果语句是 `select * from T where ID=500`，即主键查询方式，则只需要搜索ID这棵B+树；
- 如果语句是 `select * from T where k=5`，即普通索引查询方式，则需要**先搜索k索引树，得到ID的值为500，再到ID索引树搜索一次。这个过程称为回表。**

## 索引维护

新建索引之后，虽然查询性能有显著提高。但随之而来的是对索引的维护。特别是当插入数据的时候，要更新索引。因为 MySQL 会始终让这些数据保持有序。如当插入 700 的时候只需要在最后面添加数据即可。但是当插入 400 的时候，就很麻烦了，需要将 300 之后的数据依依往后挪。还有当数据在一个数据页中放不下时，这时的查询性能更糟糕。并且空间利用率也下降到 50%。这个过程被称为 “页分裂”。当页分裂越来越多时，即空间利用率达到一定的阈值时就会发生 “页合并”。

## 索引重建

- 重建主键索引

  ```mysql
  alter table T drop primary key;
  alter table T add primary key(id);
  ```

- 重建一般索引

  ```mysql
  alter table T drop index k;
  alter table T add index(k);
  ```

要注意的问题：

直接通过上面的语句重建索引有没有问题？

**重建索引的过程会创建一个新的索引，把数据按顺序插入，这样页面的利用率最高，也就是索引更紧凑、更省空间。**

但是，**重建主键的过程不合理**。不论是删除主键还是创建主键，都会将整个表重建。这样的情况下，重建一般索引也就跟着失效了。

这个时候应该使用 ` alter table T engine=InnoDB`

## 索引查询分析

```mysql
select * from T where k between 3 and 5;
```

这个查询语句，MySQL 需要执行几次树的搜索操作，会扫描多少行？

1. 在 k 索引树上找到 k=3 的记录，取得 ID = 300；
2. 再到 ID 索引树查到 ID=300 对应的 R3；
3. 在k索引树取下一个值 k=5，取得 ID=500；
4. 再回到 ID 索引树查到 ID=500 对应的 R4；
5. 在k索引树取下一个值 k=6，不满足条件，循环结束。

在这个过程中，**回到主键索引树搜索的过程，我们称为回表**。可以看到，这个查询过程读了 k 索引树的3条记录（步骤1、3和5），回表了两次（步骤2和4）。

这里有个避免回表过程的方法，那就是[覆盖索引（联合索引）](mssql-index-include.md)。

如将上述语句更改成：`select ID from T where k between 3 and 5`；只需要查 ID 的值，这个 ID 已经存储在索引树上了，所以就不需要额外的回表操作了。

**由于覆盖索引可以减少树的搜索次数，显著提升查询性能，所以使用覆盖索引是一个常用的性能优化手段。**

### 索引的最左前缀原则

当使用联合索引时，这几个索引时有顺序要求的。这个顺序规则就是 “最左前缀原则”。顺序不对索引是没有效果的。如

```mysql
index (name,k);

select name,k from T where k between 3 and 5;	// 索引有效
select k,name from T where k between 3 and 5;	// 索引无效，必须要额外新建一个 k 索引
```

### 索引下推

```mysql
mysql>
CREATE TABLE `tuser` (
  `id` int(11) NOT NULL,
  `id_card` varchar(32) DEFAULT NULL,
  `name` varchar(32) DEFAULT NULL,
  `age` int(11) DEFAULT NULL,
  `ismale` tinyint(1) DEFAULT NULL,
  PRIMARY KEY (`id`),
  KEY `id_card` (`id_card`),
  KEY `name_age` (`name`,`age`)
) ENGINE=InnoDB

select * from tuser where name like '张%' and age=10 and ismale=1;
```

![](asserts/89f74c631110cfbc83298ef27dcd6370.jpg)

首先根据最左前缀原则，只能用 “张”，找到第一个满足条件的记录 ID3。

在 MySQL 5.6之前，只能从 ID3 开始一个个回表。到主键索引上找出数据行，再对比字段值。

而 MySQL 5.6 引入的索引下推优化（index condition pushdown)， **可以在索引遍历过程中，对索引中包含的字段先做判断，直接过滤掉不满足条件的记录，减少回表次数。**

