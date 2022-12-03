CreateTime:2019-05-08 16:32:23.0

## Identity

Id核心意义： 唯一性

最好保证 趋势自增性（提高性能,参照btree+为例）


## 海量数据存贮优化（分库分区分表）

数据库连接数 有限 （分库  提高连接数）

单表性能瓶颈 （分表 减低锁粒度，降低表性能【参考map reduce思想】）

常用分表算法

     取模： 优势简单  劣势： 不好扩容 （常见 crc32 hash取模）
     hash: 好扩容  分布不均
     位移:  uid >> seed

一般都是通过自增id来进行处理

## 一些特殊业务处理

单调递增， 信息安全（不能用id来暴露）, 信息冗余（通过id就能获取一些更多信息，比如多位id组合  比如：业务号+时间+业务表+实际ID）

![](https://oscimg.oschina.net/oscnet/1c0f0593f220a76775ac1df000456ee6bb4.jpg)

UUID

XXXXXXXX-XXXX-MXXX-NXXX-XXXXXXXXXX

UUID的唯一缺陷在于生成的结果串会比较长。关于UUID这个标准使用最普遍的是微软的GUID(Globals Unique Identifiers)。在ColdFusion中可以用CreateUUID()函数很简单地生成UUID，其格式为：xxxxxxxx-xxxx- xxxx-xxxxxxxxxxxxxxxx(8-4-4-16)，其中每个 x 是 0-9 或 a-f 范围内的一个十六进制的数字。而标准的UUID格式为：xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (8-4-4-4-12)，可以从cflib 下载CreateGUID() UDF进行转换

redis: incr key(这个一般用在锁？ 限流？)

zookeeper: 持久性、临时性、顺序性节点


## 著名的分布式ID算法

雪花算法： [雪花算法-Snowflake](https://www.sohu.com/a/232008315_453160

时间戳-位移-拼接

比较上次毫秒时间，序号自增，新的一个毫秒，则置位0

毫秒数在高位，自增数列在低位，天然有序

![](http://5b0988e595225.cdn.sohucs.com/images/20180518/1af4eb1ec81d4501a757cfdc9ada72d1.jpeg)

SnowFlake所生成的ID一共分成四部分：

1. 第一位,占用1bit，其值始终是0，没有实际作用。

2. 时间戳,占用41bit，精确到毫秒，总共可以容纳约69 年的时间。

3. 工作机器id,占用10bit，其中高位5bit是数据中心ID（datacenterId），低位5bit是工作节点ID（workerId），做多可以容纳1024个节点。

4. 序列号,占用12bit，这个值在同一毫秒同一节点上从0开始不断累加，最多可以累加到4095。

SnowFlake算法的优点：

1. 生成ID时不依赖于DB，完全在内存生成，高性能高可用。

2. ID呈趋势递增，后续插入索引树的时候性能较好。

SnowFlake算法的缺点：

依赖于系统时钟的一致性。如果某台机器的系统时钟回拨，有可能造成ID冲突，或者ID乱序。闰秒，协调世界时。

## 分布式全局ID的要求：
低延迟，高可用，高并发

### 解决方式
高并发可用：
![](https://oscimg.oschina.net/oscnet/06ec466acc8bb6881176d95f253c151cab3.jpg)

还有一种解决思路：

提前批量生成，当数量少于多少之后自动再次批量生成，这样就只用从内存取（但是要考虑宕机数据恢复问题）

完全递增是很难高并发实现，在保证趋势递增基础上实现即可。


### 国内的开源项目：

[美团-leaf](https://github.com/Meituan-Dianping/Leaf)
