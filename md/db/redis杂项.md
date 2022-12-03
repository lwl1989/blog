CreateTime:2018-12-05 19:36:52.0

> 主要介绍杂项命令的使用

[主要来源 redis doc](http://redisdoc.com "主要来源 redis doc")

## config get/set

CONFIG GET 命令用于取得运行中的 Redis 服务器的配置参数(configuration parameters)，在 Redis 2.4 版本中， 有部分参数没有办法用 CONFIG GET 访问，但是在最新的 Redis 2.6 版本中，所有配置参数都已经可以用 CONFIG GET 访问了。

CONFIG GET 接受单个参数 parameter 作为搜索关键字，查找所有匹配的配置参数，其中参数和值以“键-值对”(key-value pairs)的方式排列。

比如执行 CONFIG GET s* 命令，服务器就会返回所有以 s 开头的配置参数及参数的值：

```
 1) "slave-announce-ip"
 2) ""
 3) "set-max-intset-entries"
 4) "512"
 5) "slowlog-log-slower-than"
 6) "10000"
 7) "slowlog-max-len"
 ......
```

所有被 CONFIG SET 所支持的配置参数都可以在配置文件 redis.conf 中找到，不过 CONFIG GET 和 CONFIG SET 使用的格式和 redis.conf 文件所使用的格式有以下两点不同：

10kb 、 2gb 这些在配置文件中所使用的储^^^^存^^啊^^单^^^^位缩写，不可以用在 CONFIG 命令中， CONFIG SET 的值只能通过数字值显式地设定。（ps: 为什么 储^^^^存^^啊^^单^^^^位 加一个^^^^  因为 因为 osc不让发cundan啊 天啦）

像 CONFIG SET xxx 1k 这样的命令是错误的，正确的格式是 CONFIG SET xxx 1000 。
save 选项在 redis.conf 中是用多行文字储存的，但在 CONFIG GET 命令中，它只打印一行文字。

以下是 save 选项在 redis.conf 文件中的表示：
```
save 900 1
save 300 10
save 60 10000
```
但是 CONFIG GET 命令的输出只有一行：

```
redis> CONFIG GET save
1) "save"
2) "900 1 300 10 60 10000"
```

上面 save 参数的三个值表示：在 900 秒内最少有 1 个 key 被改动，或者 300 秒内最少有 10 个 key 被改动，又或者 60 秒内最少有 1000 个 key 被改动，以上三个条件随便满足一个，就触发一次保存操作。

## dbsize

返回当前数据库的 key 的数量。

```
127.0.0.1:6379> dbsize
(integer) 7146
```

## info 

返回redis服务器详情

## MONITOR

实时打印redis接受到的命令

```
127.0.0.1:6379> MONITOR
OK
# 以第一个打印值为例
# 1378822099.421623 是时间戳
# [0 127.0.0.1:56604] 中的 0 是数据库号码， 127... 是 IP 地址和端口
# "PING" 是被执行的命令
1378822099.421623 [0 127.0.0.1:56604] "PING"
1378822105.089572 [0 127.0.0.1:56604] "SET" "msg" "hello world"
1378822109.036925 [0 127.0.0.1:56604] "SET" "number" "123"
1378822140.649496 [0 127.0.0.1:56604] "SADD" "fruits" "Apple" "Banana" "Cherry"
1378822154.117160 [0 127.0.0.1:56604] "EXPIRE" "msg" "10086"
1378822257.329412 [0 127.0.0.1:56604] "KEYS" "*"
1378822258.690131 [0 127.0.0.1:56604] "DBSIZE"
```


