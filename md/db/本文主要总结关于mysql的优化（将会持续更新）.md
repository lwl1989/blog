CreateTime:2018-08-14 15:15:46.0

### ON DUPLICATE KEY UPDATE

#### 事件背景

在阅读公司原来代码的过程中，我发现了这样一段代码:

```
$sql = "INSERT INTO {$table} ({$fields}) VALUES " . $values;
if (!empty($onDuplicate)) {
    $sql .= ' ON DUPLICATE KEY UPDATE '. $onDuplicate;
}
```

在语义的理解上，应当是索引冲突则更新原有索引数据。经过查阅资料，我总结如下：

假设业务上我们需要的就是如果存在则更新,如果不存在则新增. INSERT 中ON DUPLICATE KEY UPDATE（用redis的kv就可以很容易的实现.在MySQL中也有这样的功能）

但是这个在在使用的时候需要把关键的字段（列）设置为key ,unique key。（也就是会发生冲突的索引）

#### INSERT 中ON DUPLICATE KEY UPDATE的使用：

如果您指定了ON DUPLICATE KEY UPDATE，并且插入行后会导致在一个UNIQUE索引或PRIMARY KEY中出现重复值，则执行旧行UPDATE。


#### 栗子
```
CREATE TABLE `test_duplicate` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `a` int(11) NOT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
insert into test_duplicate (a,b,c) values(1,2,3);
```
假设我们有表如上，SQL列a被定义为UNIQUE，并且包含值1，则以下两段语句具有相同的效果：

```
mysql>INSERT INTO test_duplicate (a,b,c) VALUES (1,2,3) ON DUPLICATE KEY UPDATE c=c+1;  
mysql> select * from test_duplicate;
+----+---+---+---+
| id | a | b | c |
+----+---+---+---+
|  1 | 1 | 2 | 4 |
+----+---+---+---+

mysql>SELECT id,a,b,c from test_duplicate where a=1;
mysql>UPDATE table SET c=c+1 WHERE id=1; 

SELECT id,a,b,c from test_duplicate where a=1;
+----+---+---+---+
| id | a | b | c |
+----+---+---+---+
|  1 | 1 | 2 | 5 |
+----+---+---+---+
```
从结果可以看出来，2段SQL都都c进行了+1操作。但是insert实际上并没有进行插入数据而是进行了更新数据。

那如果，我们表内有两个可能会产生冲突的键时，又会如何呢？

```
mysql> ALTER TABLE `test_duplicate` ADD UNIQUE(`b`);
mysql> INSERT INTO test_duplicate (a,b,c) VALUES (2,3,4);


mysql> INSERT INTO test_duplicate (a,b,c) VALUES (1,3,4) ON DUPLICATE KEY UPDATE c=c+1;
Query OK, 2 rows affected (0.00 sec)

mysql> select * from test_duplicate;
+----+---+---+---+
| id | a | b | c |
+----+---+---+---+
|  1 | 1 | 2 | 6 |
|  3 | 2 | 3 | 4 |
+----+---+---+---+
```

可以看出来，同时更新了两条数据 。

那假如同一行数据，我们有两个冲突的值会产生怎么样的结果呢？
```
mysql> INSERT INTO test_duplicate (a,b,c) VALUES (1,2,4) ON DUPLICATE KEY UPDATE c=c+1;
mysql> select * from test_duplicate;
+----+---+---+---+
| id | a | b | c |
+----+---+---+---+
|  1 | 1 | 2 | 7 |
|  3 | 2 | 3 | 4 |
+----+---+---+---+

```

因此，我们在设计表的时候，应该尽量避免多冲突值得存在，如果实在避免不了，我们可以使用values方法获取本次提交的值。VALUES()函数只在INSERT...UPDATE语句中有意义，其它时候会返回NULL。

```
mysql> INSERT INTO test_duplicate (a,b,c) VALUES (1,2,4) ON DUPLICATE KEY UPDATE c=values(c);
mysql> select * from test_duplicate;
+----+---+---+---+
| id | a | b | c |
+----+---+---+---+
|  1 | 1 | 2 | 4 |
|  3 | 2 | 3 | 4 |
+----+---+---+---+
```
应该一般需求都是要达到这个结果

> 需要注意的是，在事务中，只有SELECT ... FOR UPDATE 或LOCK IN SHARE MODE 同一笔数据时会等待其它事务结束后才执行，一般SELECT ... 则不受此影响。拿上面的实例来说，当我执行select status from t_goods where id=1 for update;后。我在另外的事务中如果再次执行select status from t_goods where id=1 for update;则第二个事务会一直等待第一个事务的提交，此时第二个查询处于阻塞的状态，但是如果我是在第二个事务中执行select status from t_goods where id=1;则能正常查询出数据，不会受第一个事务的影响。

### 关于老的数据库密码设置

刚入职的时候，我编译了自己的docker是php7环境的，然后无法支持mysql只支持mysqlnd作为pdo驱动。于是乎，错误来了

> "SQLSTATE[HY000] [2000] mysqlnd cannot connect to MySQL 4.1+ using the old insecure authentication. Please use an administration tool to reset your password with the command SET PASSWORD = PASSWORD('your_existing_password'). This will store a new, and more secure, hash value in mysql.user. If this user is used in other scripts executed by PHP 5.2 or earlier you might need to remove the old-passwords flag from your my.cnf file

在google遨游了很久，也没找到解决方式，然后，也尝试着装mysql然后编译PHP的时候使用mysql的头文件尝试修改mysqlnd的方式也没有成功。

网上大部分答案都需要登入mysql服务器去改my.cnf。

最后终于搞定了,将他记录下来，原来可以临时修改会话的密码长度然后重设。

```
mysql> SELECT user, Length(`Password`) FROM  `mysql`.`user`;
+----------------+--------------------+
| user           | Length(`Password`) |
+----------------+--------------------+
| root           |                 16 |
| root           |                  0 |
| root           |                  0 |
|                |                  0 |
|                |                  0 |
| root           |                 16 |
| test           |                 16 |
| club_star_user |                 16 |
| club_star_user |                 16 |
| wenlong11      |                 16 |
+----------------+--------------------+
```

```
mysql> SET SESSION old_passwords = 0;
Query OK, 0 rows affected (0.00 sec)
```

```
mysql> SELECT @@global.old_passwords, @@session.old_passwords, Length(PASSWORD('abc'));
```

```
mysql> UPDATE mysql.user SET Password = PASSWORD('123456') WHERE user = 'xxxx';
Query OK, 1 row affected (0.00 sec)
Rows matched: 1  Changed: 1  Warnings: 0

mysql> flush privileges;
Query OK, 0 rows affected (0.00 sec)

mysql> SELECT user, Length(`Password`) FROM  `mysql`.`user`;
+----------------+--------------------+
| user           | Length(`Password`) |
+----------------+--------------------+
| root           |                 16 |
| root           |                  0 |
| root           |                  0 |
|                |                  0 |
|                |                  0 |
| root           |                 16 |
| test           |                 16 |
| club_star_user |                 16 |
| club_star_user |                 16 |
| wenlong11      |                 41 |
+----------------+--------------------+

```

可以观察到 密码长度终于变成41的新版的长度了



### order by 排序不准

mysql排序  假如对很多值相等的值进行order  分页 会产生乱序 导致数据重复

	select * from where a = 3 and b = 5 order by c desc
	
	select * from where a = 3 and b =5 order by c desc, d desc
	

解决方式：
1、尽量不要使用这种字段排序
2、如果业务需求，将其修改成多种混合  如  order field  desc =>    order field desc,  uniqueField desc
        确保结果不会混乱
### MySQL Explain

示例: explain select * from tablename;

将会得出结果查询详情，而不是结果集

expain出来的信息有10列，分别是id、select_type、table、type、possible_keys、key、key_len、ref、rows、Extra

### force index 

在某些情况下，mysql推荐的索引并不是我们最想用的业务（根据业务需求）

这个时候，我们可以将自己确定的索引进行设置，保证本条SQL强制走索引

	select * from $table_name force index(index_name) where condition  limit number
	
[测试性能比较](https://blog.csdn.net/bruce128/article/details/46777567 "测试性能比较")

8000W 数据，不用force index 200s都未查询完毕

加了之后，1S左右完成

执行explain,发现这个sql扫描了8000W条记录到磁盘上。然后再进行筛选。type=index说明整个索引树都被扫描了，效果显然不理想。

### ignore index

对应的，在某些情况下我们确定了不需要某个索引

这个时候，我们可以将此索引忽略，保证本条SQL不遍历这个索引

