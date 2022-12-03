CreateTime:2021-02-23 18:31:49.0

# binlog

mysql为了保证事务的ACID(atomicity,consistency,isolation,durability)，用了几种日志做配合处理，分别为binglog（二进制日志）、redolog（重做日志）、undolog（回滚日志）。

重做日志（redo log）

	确保事务的持久性。防止在发生故障的时间点，尚有脏页未写入磁盘，在重启mysql服务的时候，根据redo log进行重做，从而达到事务的持久性这一特性。

回滚日志（undo log）

	保存了事务发生之前的数据的一个版本，可以用于回滚，同时可以提供多版本并发控制下的读（MVCC），也即非锁定读

二进制日志（binlog）：

	用于复制，在主从复制中，从库利用主库上的binlog进行重播，实现主从同步。 用于数据库的基于时间点的还原。

我们这里的重点，就是binlog了。

# 同步机制

![](https://oscimg.oschina.net/oscnet/up-5af8f2407d53b1c22b380928f7cac40e9e4.png)

通过图我们可以看出，binlog同步分为6个步骤：

1. master开启binlog日志（数据改变会产生日志）
2. slave连接mater，开启同步（前提，同名db必须存在，假如数据不为空，已有数据必须一致）
3. master数据变化产生binglog，通过binglog dump线程通知给slave.
4. slave同步写入relay log。
5. slave执行收到的sql日志（此时已经执行完毕）。
6. slave执行写本地binlog。

binlog同步分为3种同步模式：
1. STATEMENT模式（SBR）

        每一条会修改数据的sql语句会记录到binlog中。优点是并不需要记录每一条sql语句和每一行的数据变化，减少了binlog日志量，节约IO，提高性能。缺点是在某些情况下会导致master-slave中的数据不一致(如sleep()函数， last_insert_id()，以及user-defined functions(udf)等会出现问题)

2. ROW模式（RBR）

        不记录每条sql语句的上下文信息，仅需记录哪条数据被修改了，修改成什么样了。而且不会出现某些特定情况下的存储过程、或function、或trigger的调用和触发无法被正确复制的问题。缺点是会产生大量的日志，尤其是alter table的时候会让日志暴涨。

3. MIXED模式（MBR）

        以上两种模式的混合使用，一般的复制使用STATEMENT模式保存binlog，对于STATEMENT模式无法复制的操作使用ROW模式保存binlog，MySQL会根据执行的SQL语句选择日志保存方式。

# 步骤

### master开启binlog
```
binlog_format           = MIXED                         //binlog日志格式，mysql默认采用statement，建议使用mixed
log-bin                 = /data/mysql/mysql-bin.log     //binlog日志文件
expire_logs_days        = 7                             //binlog过期清理时间
max_binlog_size         = 100m                          //binlog每个日志文件大小
binlog_cache_size       = 4m                            //binlog缓存大小
max_binlog_cache_size   = 512m                          //最大binlog缓存大小
```

### slave连接

```
server-id=2
master-host=xxx
master-user=username
master-password=xxxx
master-port=xxxx
replicate-do-db=dbname
......

# 开启中继日志
relay-log=relay-log
relay-log-index=/data/mysql/relay-log.index
server-id=2
innodb_file_per_table=ON #innodb_file_per_table=ON的情况下，默认创建出来的ibd文件的格式是Barracuda，在这个文件格式下innodb数据行的格式就可以设置为compressed 或 dynamic 格式了。compressed 提供压缩功能节约空间，dynamic能优化对blob,text这样的数据类型的存储以提升性能。5.7+才支持
skip_name_resolve=ON  #跳过主机检测
```

### binlog dump theard

主库在接收到从库发送的COM_BINLOG_DUMP_GTID命令后，会调用com_binlog_dump_gtid函数处理从库拉取binlog的请求。主要的执行逻辑如下：

1. 获取从库发送的binlog相关信息，包括server_id，binlog名称，binlog位置，binlog大小，gtid信息等等。
2. 检查是否已经存在与该从库关联的binlog dump线程，如果存在，结束该binlog dump线程。为什么会已经存在binlog dump线程？在某些场景下，比如从库io线程停止，这时主库的binlog dump线程正好在等待binlog更新，即等待主库写入数据，如果主库一直没有写入数据，dump线程就会等待很长时间，如果这时从库io线程重连到主库，就会发现主库已经存在与该从库对应的dump线程。所以主库在处理从库binlog dump请求时，先检查是否已经存在dump线程。
3. 调用mysql_binlog_send函数，向从库发送binlog。这个函数内部实际是通过一个C++类Binlog_sender来实现的，该类在源码文件sql/rpl_binlog_sender.h中定义，调用该类的run成员函数来发送binlog。
4. Binlog_sender类的run成员函数，主要逻辑是通过多个while嵌套循环，依次读取binlog文件，binlog文件中的event，将event发送给从库。如果event已经在从库中应用，则忽略该event。当读到最新的binlog时，如果所有event都已经发送完成，那么线程会等待binlog更新事件update_cond，有新的binlog event写入后，会广播通知所有等待update_cond事件的线程开始工作，也包括dump线程。dump线程在等待update_cond事件时有一个超时时间，这个时间就是master_heartbeat_period，即主库dump线程与从库io线程的心跳时间间隔，这个值在从库执行change master 时设置，启动io线程时把该值传递给主库，主库dump线程等待update_cond超时后，将会给从库发送一个heartbeat event，之后会继续等待update_cond事件。上述过程会一直循环，直到dump线程被kill或者遇到其他错误。
5. 当执行逻辑从Binlog_sender类内部的while循环退出，紧接着会调用unregister_slave函数注销从库的注册。这个时候在主库上执行show slave hosts，就会发现从库的信息已经没有了。

### slave relaylog


![](https://oscimg.oschina.net/oscnet/up-3564e30236590f93a249cdb6c175f97a1d9.png)

relay-log的结构和binlog非常相似，只不过他多了一个master.info和relay-log.info的文件。

master.info记录了上一次读取到master同步过来的binlog的位置，以及连接master和启动复制必须的所有信息。

relay-log.info记录了文件复制的进度，下一个事件从什么位置开始，由sql线程负责更新。

##### relay-log无法自动删除

背景：

在运维一个mysql实例时，发现其数据目录下的relay-log 长期没有删除，已经堆积了几十个relay-log。 然而其他作为Slave服务器实例却没有这种情况。

现象分析：

通过收集到的信息，综合分析后发现relay-log无法自动删除和以下原因有关。

- 该实例原先是一个Slave:导致relay-log 和 relay-log.index的存在
- 该实例目前已经不是Slave:由于没有了IO-Thread，导致relay-log-purge 没有起作用（ 这也是其他Slave实例没有这种情况的原因，因为IO-thread会做自动rotate操作）。
- 该实例每天会进行日常备份:Flush logs的存在，导致每天会生成一个relay-log
- 该实例没有配置expire-logs-days:导致flush logs时，也不会做relay-log清除
- 简而言之就是： 一个实例如果之前是Slave，而之后停用了（stop slave），且没有配置expire-logs-days的情况下，会出现relay-log堆积的情况。

Binary Log rotate机制：

- Rotate：每一条binary log写入完成后，都会判断当前文件是否超过max_binlog_size ，如果超过则自动生成一个binlog file
- Delete：expire-logs-days 只在 实例启动时 和 flush logs 时判断，如果文件访问时间早于设定值，则purge file

Relay Log rotate 机制：

- Rotate：每从Master fetch一个events后，判断当前文件是否超过max_relay_log_size 如果超过则自动生成一个新的relay-log-file
- Delete： purge-relay-log 在SQL Thread每执行完一个events时判断，如果该relay-log 已经不再需要则自动删除
- Delete： expire-logs-days 只在 实例启动时 和 flush logs 时判断，如果文件访问时间早于设定值，则purge file （同Binlog file） (updated: expire-logs-days和relaylog的purge没有关系)

```
因此还是建议配置 expire-logs-days ， 否则当我们的外部脚本因意外而停止时，还能有一层保障。
因此建议当slave不再使用时，通过reset slave来取消relaylog，以免出现relay-log堆积的情况。
```

###  salve sql exe

这个执行步骤和一般执行sql没有太大区别

### slave wirte binlog

这个执行步骤和master一致

# 各种同步模式的优缺点

### SBR 的优点：

历史悠久，技术成熟：
- binlog文件较小
- binlog中包含了所有数据库更改信息，可以据此来审核数据库的安全等情况
- binlog可以用于实时的还原，而不仅仅用于复制

主从版本可以不一样，从服务器版本可以比主服务器版本高


### SBR 的缺点：

不是所有的UPDATE语句都能被复制，尤其是包含不确定操作的时候。调用具有不确定因素的 UDF 时复制也可能出问题，使用以下函数的语句也无法被复制：
* LOAD_FILE()
* UUID()
* USER()
* FOUND_ROWS()
* SYSDATE() (除非启动时启用了 --sysdate-is-now 选项)

INSERT ... SELECT 会产生比 RBR 更多的行级锁

复制需要进行全表扫描(WHERE 语句中没有使用到索引)的 UPDATE 时，需要比 RBR 请求更多的行级锁。对于有 AUTO_INCREMENT 字段的 InnoDB表而言，INSERT 语句会阻塞其他 INSERT 语句。对于一些复杂的语句，在从服务器上的耗资源情况会更严重，而 RBR 模式下，只会对那个发生变化的记录产生影响。存储函数(不是存储过程)在被调用的同时也会执行一次 NOW() 函数，这个可以说是坏事也可能是好事，确定了的 UDF 也需要在从服务器上执行，数据表必须几乎和主服务器保持一致才行，否则可能会导致复制出错，执行复杂语句如果出错的话，会消耗更多资源

### RBR 的优点：

任何情况都可以被复制，这对复制来说是最安全可靠的，和其他大多数数据库系统的复制技术一样，多数情况下，从服务器上的表如果有主键的话，复制就会快了很多，复制以下几种语句时的行锁更少：
* INSERT ... SELECT
* 包含 AUTO_INCREMENT 字段的 INSERT
* 没有附带条件或者并没有修改很多记录的 UPDATE 或 DELETE 语句

执行 INSERT，UPDATE，DELETE 语句时锁更少，从服务器上采用多线程来执行复制成为可能

### RBR 的缺点：

binlog 大了很多，复杂的回滚时 binlog 中会包含大量的数据，主服务器上执行 UPDATE 语句时，所有发生变化的记录都会写到 binlog 中，而 SBR 只会写一次，这会导致频繁发生 binlog 的并发写问题，UDF 产生的大 BLOB 值会导致复制变慢，无法从 binlog 中看到都复制了写什么语句，当在非事务表上执行一段堆积的SQL语句时，最好采用 SBR 模式，否则很容易导致主从服务器的数据不一致情况发生

	另外，针对系统库 mysql 里面的表发生变化时的处理规则如下：
	如果是采用 INSERT，UPDATE，DELETE 直接操作表的情况，则日志格式根据 binlog_format 的设定而记录
	如果是采用 GRANT，REVOKE，SET PASSWORD 等管理语句来做的话，那么无论如何都采用 SBR 模式记录
	注：采用 RBR 模式后，能解决很多原先出现的主键重复问题。

