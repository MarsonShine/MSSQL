# MYSQL 字符集

MYSQL 推荐使用 `UTF8MB4` 字符集，这样就可以将那些网络 emoji 表情转码成字符集存储到 mysql 字段中

```mysql
select cast(0x61 as char(1) charset utf8mb4) as ch; -- 将16禁止转换成字符 输出 'a'
select cast(0xF09F988E as char(1) charset utf8mb4) as ch; -- 将 0xF09F988E 转成 emoji，注意只能在 utf8mb4 下才能正常显示
```

我们也可以手动显示所有的字符集：

```mysql
show charset like 'utf8%'; -- 显示所有utf8开头的字符集
show collation like 'utf8mb4%'; -- 显示所有 utf8mb4 开头的字符集排序规则

...
utf8mb4_general_ci	utf8mb4	45	Yes	Yes	1
utf8mb4_bin	utf8mb4	46		Yes	1
utf8mb4_unicode_ci	utf8mb4	224		Yes	8
utf8mb4_icelandic_ci	utf8mb4	225		Yes	8
utf8mb4_latvian_ci	utf8mb4	226		Yes	8
utf8mb4_romanian_ci	utf8mb4	227		Yes	8
...
-- 上面 ci 结尾的代表不区分大小（Case Insentive）; cs 区分大小写
```

## 如何修改已有表的字符集

```mysql
alter table my_table charset utf8mb4;
```

这上面有个问题，就是对于已有的数据对应的字段，它以前是属于什么字符集，那么即使你改了还是以前的字符集类型不会变。这时我们可以用 `convert to` 执行：

```mysql
alter table my_table convert to charset utf8mb4;
```

## 如何设置 bool 类型的字段

比如我们在设计男女或是其它状态带有正反两面意思的字段时，我们可以选择 `int` 也可以选择 `tinyint` 等数字类型。但是其实也有问题的：

- 表达意思不清晰：在具体存储时，0 代表男还是女。每个业务选择的方式可能都有不同。这种我们实际中大部分应该是在备注上进行说明，但是这并不能没有解决根本问题。
- 脏数据：因为是数字类型，所以我们可以存储除 0/1 之外的数字，一旦程序没有写好，让这些数字进来那么就会引起数据混乱，后期清理代价就非常大了。

### ENUM 枚举类型

其实 MYSQL 有枚举类型 `enum`，这个类型允许插入有限个的值，这个值在 MYSQL 内部做了映射。在 `SET SQL_MODE = 'STRICT_TRANS_TABLES'` 严格模式下，插入非选定值是会报错的。

设置 enum 类型：

```mysql
alter table users modify sex enum('Man','Women') collate utf8mb4_general_ci default null; -- Man = 1, Women = 2, 0 = null
```

注意：enum 类型并非 SQL 标准的数据类型，它是 MYSQL 独有的。所以在选择 enum 字段的时候要慎重。虽然 enum 对可读性和脏数据有明显的好处。但是同样也伴随很多缺陷。最明显的就是它不是 SQL 标准字段类型；如果出现跨数据库迁移，那就很麻烦了。还有对应的 ORM 对 enum 类型支持也不够好。更多缺陷可参考：https://www.zhihu.com/question/404422255

