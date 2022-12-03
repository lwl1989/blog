CreateTime:2021-02-25 15:52:16.0

# replica set

副本集形式，类似于mysql master slave，这里成为primary 和 secondary，具有选举机制，最低3台，1主2从。

![](https://oscimg.oschina.net/oscnet/up-fa951a5d49736dca2b5b50adf0664996d25.png)

复制主要用于备份、灾难恢复和读写分离。一个Replica Set就是一组mongod实例。Replica Set中的Primary接收所有的写操作，Secondaries从Primary复制操作然后应用到自己的data set。

### 心跳检测

在replica sets结构中，节点之间会各自向其它节点发送一个心跳检测请求。在正常情况下，其它节点会返回一个包含自身信息的回复包，回复包中主要包括了下面一些信息：它们现在是什么角色（primary 还是 secondary)，它们是否能够在必要的时候成为 primary，以及他们当前时钟时间等等。

![](https://oscimg.oschina.net/oscnet/up-2d447537332f29ffed4b17d7f70da5132bb.png)

节点在收到回复包后，会用这些信息更新自己的一个状态映射表。

当primary节点映射表发生了变化，那么它会那自己是否还能和集群中大多数节点进行通信，如果不能与大多数节点通信，那么它会把自己从 primary 降级为 secondary。

### 降级

在节点从 primary 降级为 secondary 的过程中，会有一些问题出现。在 MongoDB 中，写操作默认是通过 fire-and-forget 的模式来进行的，也就是说写操作通常不关心是否成功，发完请求后客户端就认为成功了。但如果这时候 primary 进行降级操作，那么客户端并不知道这时候 primary 已经降级成为 secondary 了，客户端可能还会将后续的写操作发送给这个节点。这时候刚刚降级的这个 secondary 可以发送一个包说“我已经不是 primary 了”，但是我们上面说过了，客户端根本就无视你这个包。所以客户端根本不知道这次写入已经失败了。

对于这个问题，你可能会说"那我们每次都使用安全写入不就行了"（安全写入意思是说等待服务器返回成功后客户端才认为写成功了），但是很明显，这非常不靠谱。所以我们的做法是，在一个 primary 降级成为 secondary 后，它会将原来的所有连接关闭。这样客户端在下一次写入的时候就会出现 socket 错误。而客户端在发现这个错误之后，就会重新向集群获取新的 primary 的地址，并将后续的写操作都往新的服务器上写入。

### 选举

选举是一个获取多数票的过程，具有一定网络io耗时。

```
而当X发现现在需要一个 primary 并且自己又正好可以充当时.
它就会发起一轮选举：X节点会向Y、Z节点各发起一个请求包:
    告知他们,我认为我可以接管 primary 的角色，你们觉得怎么样？

其他节点在收到通知时，他们会进行下面几项检测：
    集群中是否有一个primary了？
    他们自己的数据是否比X节点更新？
    他们自己的数据是否比X节点更新？
一旦其中某一项不满足，pass.

如果都满足，会反馈给X。X会告诉其他节点，我是primary。其他节点给予回馈（投票）说同意，X才正式升级为primary。

注意：这个投票过程，有30秒间隔，单节点30秒内不能重复进行。

投票规则： 如果没有人投反对票，并且赞成票的比例过半，那么本轮选举对象就能够成为 primary。（很民主啊，人人参与~）
```

# shard(分区)

![](https://oscimg.oschina.net/oscnet/up-51ba31b29f62379f2037a24a1a41521cf35.png)

mongodb的分区功能其实比较简单，就是根据数据对元素进行分区。只是这个逻辑可以自定义而已。

和普通的replica set集群比起来，有什么不同呢？

1. 增加了route
2. 增加了config server

![](https://oscimg.oschina.net/oscnet/up-5d71b65a4c271575b2054cbb8a478aac048.png)

### 步骤

```
1. 开启config

	./bin/mongod --fork --dbpath data/config/ --logpath log/config.log –port 10000

2. 开启router

	./bin/mongos --port 20000 --configdb 192.168.1.13:10000 --logpath log/mongos.log  --fork

3. 启动各个分片服务

	./bin/mongod --dbpath data/shard1/ --logpath log/shard1.log  --fork --port 27017
	./bin/mongod --dbpath data/shard2/ --logpath log/shard1.log  --fork --port 27018
	./bin/mongod --dbpath data/shard3/ --logpath log/shard1.log  --fork --port 27019
	./bin/mongod --dbpath data/shard4/ --logpath log/shard1.log  --fork --port 27020

4. 添加分片

	./bin/mongo --port 20000
	db.runCommand({addshard:"192.168.1.13:27017",allowLocal:true })
	db.runCommand({addshard:"192.168.1.13:27018",allowLocal:true })
	db.runCommand({addshard:"192.168.1.13:27019",allowLocal:true })
	db.runCommand({addshard:"192.168.1.13:27020",allowLocal:true })

5. 默认分区规则是根据_id来的，我们也可以自定义去配置。

	sh.shardCollection('dbName.colletionName',{keyName:1});

6. 删除分片

	db.runCommand({"removeshard":"192.168.1.13:27020"})
移除分片需要一个过程，MongoDB会把移除的片上的数据（块）挪到其他片上，会造成一定耗时，要注意业务场景。

7. 管理分片

	db.shards.find()
	{ "_id" : "shard0000", "host" : "192.168.1.13:27017" }
	{ "_id" : "shard0000", "host" : "192.168.1.13:27018" }
	{ "_id" : "shard0000", "host" : "192.168.1.13:27019" }
	{ "_id" : "shard0000", "host" : "192.168.1.13:27020" }
```

### 分片片键

mongodb分片是分布式存储的，因此，片键的定义对整个性能的影响巨大。所幸，mongodb是可以自定义片键的。

好片键的要素：

	1. 读和写的分布
	2. 数据块的大小
	3. 每个查询命中的分片数目

从设计角度来说，其实和我们常用的mysql的逻辑分表很像。

分片集群数据分布方式：

	基于范围
	基于Hash
	基于zone/tag

分片集群数据分布方式-基于范围(最常用，适合一般性业务逻辑)：
|   |   |
| ------------ | ------------ |
| 片键范围查询性能好  | 数据分布可能不均匀 |
|优化读|容易有热点|
|   |   |


分片集群数据分布方式-基于哈希（适用:日志，物联网等高并发场景）：
|   |   |
| ------------ | ------------ |
| 数据分布均匀，写优化  | 范围查询效率低 |
|   |   |

# 读写（转图）

![](https://oscimg.oschina.net/oscnet/up-f5d2bfebd059ef681ebeb8c068623002877.png)
![](https://oscimg.oschina.net/oscnet/up-40a038595de4b5fe1418606cdc43a2c2da9.png)
![](https://oscimg.oschina.net/oscnet/up-a86870bbf02194a4b062de8ae5aa7c3c268.png)

分析：

1. 如果不能命中片键，范围查询将会扫描全表（性能极地，损耗极大），结果集将在route层聚集（类似于hadoop的map reduce）
2. 需要合理提前规划片键，否则后期移动数据将需要复杂处理。
3. 如果集群在_id上进行了分片，则无法再在其他字段上建立唯一索引。
