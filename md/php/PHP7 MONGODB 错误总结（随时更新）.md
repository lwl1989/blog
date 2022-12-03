CreateTime:2016-02-22 12:06:45.0

### 1.bad query: BadValue $in needs an array
产生原因，数组下标不是有序从0开始的数字
解决方案 array_values($array)

![输入图片说明](https://static.oschina.net/uploads/img/201602/22114723_X4Zs.png "在这里输入图片标题")

### 2.auth用户添加

    db.addUser('sa','sa');
    2016-02-25T09:56:17.537+0800 E QUERY    [thread1] TypeError: db.addUser is not a                                                               function :
    @(shell):1:1

产生原因:

    V3以上的版本,弃用了 db.addUser 

解决方案：

    mongodb使用db.createUser() db.createUser({user:"sa",pwd:"sa",roles:[ {role:"readWrite",db:"test"} ]}); 

运行结果： 

    Successfully added user: { "user" : "root", "roles" : [ { "role" : "readWrite", "db" : "paintmore" } ] }

详细语法参见：
    https://docs.mongodb.org/manual/reference/method/db.createUser/#create-administrative-user-with-roles

### 3.集群配置(填写好URI自动实现主从配置，网络内存越好的节点自动被推荐为主节点，很想hdfs的master+standby,但是又稍有不同)

final public MongoDB\Driver\Manager::__construct ( string $uri [, array $options [, array $driverOptions ]] )

$uri:

     mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]] 
规格属性集合
example:

    mongodb://db1.example.net,db2.example.net:2500/?replicaSet=test&connectTimeoutMS=300000  

MongoDB\Driver\Manager::getServers — Return the servers to which this manager is connected
MongoDB\Driver\Manager::selectServer — Select a server matching a read preference


http://my.oschina.net/jsk/blog/644287   可以参考

### 4.oom解決方案（2018-3-23）

前段时间公司测试机mongodb经常crash,去看日志发现是oom-killer.(环境是docker mongdob shard replicate)

既然是oom，那肯定是由于申请内存不够导致。

详细关于OOM可以看[知乎oom](https://www.zhihu.com/question/21972130),知乎的东西当然是仁者见仁，智者见智了。

找到一个方案是http://blog.fens.me/linux-upstart-mongodb/ ，但是他是用自动重启的方案去搞定，我觉得会造成数据丢失，因此pass。

由于测试机的log数据太多，因此我删除了大量的log数据，但是发现一段时间后又继续发生oom。只好再排查，去官网看oom的解决方案，发现是一个例子：https://jira.mongodb.org/browse/SERVER-22000.  找到了一个关键字 WiredTiger。

MongoDB3.4开始，WiredTiger将会内部缓存将会占用以下（较大的一个）内存空间：50%的内存减去1GB  或者 256MB
但是由于我们测试机是aws的最低配置1G内存，而且我还跑了2个及以上的docker（用docker隔离的mongodb实例，但是并没有对docker实例设置内存限制，因此，每个docker都在抢宿主机内存，这时候当某个分片查询过多，缓存数据导致oom）。

推荐的方案解决：
    
    设置storage.wiredTiger.engineConfig.cacheSizeGB，限制的是WiredTiger内部缓存占用内存大小。因为使用文件系统缓存的原因，操作系统会使用剩余的全部内存空间（所以跑着大业务量的mongod的机器内存使用总会100%）。这个设定（WiredTiger内部缓存默认值）的前提是一个机器上只运行一个mongod实例，如果你的机器上运行了多个mongod实例的话，应该把这个配置相对调低。生产环境不推荐一个机器上启动多个mongod进程（因此我们测试环境就被这个坑了，如果是在容器中运行mongod实例，单个容器往往没有权限使用全部内存，这时候需要把storage.wiredTiger.engineConfig.cacheSizeGB的值设的小于容器可使用的内存空间大小）。

    
然后，我们发现其实我们测试机数据并不多，为何会占用这么多内存。（log我们全部停止了，为了模拟线上）。最后查找各种表，找到了商品表。

因为商品的属性过于复杂，因此设计了一个mongodb的聚合表（更优方案其实可以用ES,但是从成本和维护角度考虑，最后选用了mongodb）

表结构如下:

     { "_id" : "", "goods_id" : "2", "relation" : { ...... }, "goods" : {......}, "sku" : [   { ...... } ],"attr" : [ {  ......  }], "intro" : {  ...... },  "pick" : {   "courier" : "1",      "store_pick" : "1"  },"time" : { ...... },"images" : [  ......] }

商品表内Intro是长文本，mysql内存的是text。

采用优化方案是:
    
    在dao层进行处理，假如增删改查字段有intro，都从mysql读,不缓存到mongodb的内存加速中。

采取这次优化之后，果然，mongodb再也没有crash了！



