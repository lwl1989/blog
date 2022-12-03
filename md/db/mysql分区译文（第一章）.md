CreateTime:2018-05-18 14:08:14.0

[原文地址](https://dev.mysql.com/doc/refman/5.7/en/partitioning.html)

> 请注意
>
> mysql通用的分区程序已被弃用了(5.7.17)，并且将会在mysql8中完全移除。
> 当你为表提供自定义的（本地的）分区方法时，请修改表的驱动（只支持INNODB和NDB ）
> 
> 使用非本地的分区方法时，将会导致一个 ER_WARN_DEPRECATED_SYNTAX 错误发生。在版本为5.7.17-5.7.20~的mysql中，mysql会自动检查使用了非本地分区方法的表，并且将详情写入日志。若想禁用这个检查步骤，请使用(mysql.cnf中加入参数)：

    disable-partition-engine-check=1

> 在mysql5.7.20之后的版本中，这个检查步骤不会执行了。在上述版本之后，如果你还想进行检查，你必须设置 --disable-partition-engine-check=false,以便系统采取默认检查器进行检查。
>
> 如果你要将数据迁移到MYSQL8，则一定要将非本地分区修改成本地分区或者关闭掉分区方案。比如，修改数据库引擎为支持分区的引擎：
    
    ALTER TABLE table_name ENGINE = INNODB;

你可以执行SQL（SHOW PLUGINS;）来获取你的mysql哪些引擎支持分区：

```
mysql> SHOW PLUGINS;

+------------+----------+----------------+---------+---------+
| Name       | Status   | Type           | Library | License |
+------------+----------+----------------+---------+---------+
| binlog     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| partition  | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| ARCHIVE    | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| BLACKHOLE  | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| CSV        | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| FEDERATED  | DISABLED | STORAGE ENGINE | NULL    | GPL     |
| MEMORY     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| InnoDB     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| MRG_MYISAM | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| MyISAM     | ACTIVE   | STORAGE ENGINE | NULL    | GPL     |
| ndbcluster | DISABLED | STORAGE ENGINE | NULL    | GPL     |
+------------+----------+----------------+---------+---------+
11 rows in set (0.00 sec)
```
也可以执行SQL来判断是否支持

```
mysql> SELECT PLUGIN_NAME as Name, PLUGIN_VERSION as Version,PLUGIN_STATUS as Status FROM INFORMATION_SCHEMA.PLUGINS WHERE PLUGIN_TYPE='STORAGE ENGINE';

+--------------------+---------+--------+
| Name               | Version | Status |
+--------------------+---------+--------+
| binlog             | 1.0     | ACTIVE |
| CSV                | 1.0     | ACTIVE |
| MEMORY             | 1.0     | ACTIVE |
| MRG_MYISAM         | 1.0     | ACTIVE |
| MyISAM             | 1.0     | ACTIVE |
| PERFORMANCE_SCHEMA | 0.1     | ACTIVE |
| BLACKHOLE          | 1.0     | ACTIVE |
| ARCHIVE            | 3.0     | ACTIVE |
| InnoDB             | 5.7     | ACTIVE |
| partition          | 1.0     | ACTIVE |
+--------------------+---------+--------+
10 rows in set (0.00 sec)
```

在结果集中，如果你没有看到partition列或者partition的status不为activite，那么说明你的mysql版本并不支持分区操作。

Oracle提供的MYSQL5.7社区版可执行文件支持分区。有关MySQL企业版二进制文件中提供的分区支持的信息，请参见[MySQL企业版](https://dev.mysql.com/doc/refman/5.7/en/mysql-enterprise.html)。

如果你要从源码编译MYSQL，请在编译检查时候加入参数 -DWITH_PARTITION_STORAGE_ENGINE。
有关更多信息，请参见从[源代码安装MySQL](https://dev.mysql.com/doc/refman/5.7/en/source-installation.html)。

如果你使用的是二进制（可执行）的MYSQL，那就不需要再额外打开分区支持了，默认是开启的。如果你想禁用分区支持，请在my.cnf中加入--skip-partition。当你禁用了分区功能之后，你会发现现有已经分区过的表和数据你还是能看到并且可以删除的（虽然不建议这样做），但是，你并无法进行修改和新增了。

[第二章](https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning-overview.html) 我们将会介绍分区的概念和如何分区。

[第三章](https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning-types.html)  MySQL支持几种类型的分区和子分区（3.6 子分区）

[第四章](https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning-management.html) 将介绍如何更改现有分区（增删改，4.4 分区维护命令）


INFORMATION_SCHEMA数据库中的分区表提供了关于分区和分区表的信息。有关更多信息，请参见INFORMATION_SCHEMA分区表;对于一些针对此表的查询示例，请参见第3.7节“MySQL分区如何处理NULL”。

> 第五章呢？？？mysql文档是不是逗我呢。。。好吧。。继续

[第六章](https://dev.mysql.com/doc/mysql-partitioning-excerpt/5.7/en/partitioning-limitations.html) 说明一些MySQL 5.7中分区的已知问题

其他的帮助资料：

[MySQL分区论坛](https://forums.mysql.com/list.php?106)

[Mikael Ronström的BLOG](http://mikaelronstrom.blogspot.com/)

[PlanetMySQL](https://planet.mysql.com/)

Mysql5.7的可执行文件可在 [http://dev.mysql.com/downloads/mysql/5.7.html](http://dev.mysql.com/downloads/mysql/5.7.html)进行下载，但是，如果想获取最新特性和已知的BUG修复，你需要到mysql的github仓库进行下载后编译（打开分区参照上方）,更多编译信息和参数，参照[MYSQL编译安装](https://dev.mysql.com/doc/refman/5.7/en/source-installation.html)。如果在编写支持分区的MySQL 5.7编译的时候遇到问题，请在[MySQL分区论坛](https://forums.mysql.com/list.php?106)寻求帮助。