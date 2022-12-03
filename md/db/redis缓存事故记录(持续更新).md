CreateTime:2018-10-08 10:34:31.0

> 善于总结，才能更快进步

通常，我们对高并发的数据都会进行缓存，而且为了防止缓存过大，通常我们都会把缓存设置一个超时时间，并且会有cache miss机制。本文，我记录一下错误的缓存机制引起的BUG。

## 案例1

### 起因

好好的一个国庆，自己完全没歇停，让我给毁了。线上一次cache miss导致缓存数据错误，便一直在查因。然后重写代码、测试、上线。emmm……

### 直接看代码

当然是伪代码了

```
cache = new cache();
data = cache.getData();

if(isempty(data)) {
    data = getDataFromResource()
    if(!isempty(data)) {
        cache.setData(data)
    }
}
```

看上去没错哈，一般我们处理缓存的确是用这个步骤:

1.  读取缓存
2.  若cache miss(超时、网络原因)，从数据源读取缓存
3.  重新设置缓存

正常来说，这样的确是没问题的。

但是，请接着往下看。

资源类大致是这样的

```
//上文getDataFromResource() 就是本类中读取数据
class resource{
    private static connection = new Connection();
    public static getConnection() {
        return connection;
    }

    public getData() {
        try{
            //todo:do anythings
            data = connection.get();
            return data;
        }catch(e){
            return null;
        }
    }
}
```

而我的缓存类是基于资源类的

```
class cache extends resource {

}
```

就是说，我缓存类依赖的连接资源，也是我原始资源的来源。

### 事故原因

当其中某次请求发生错误的时候（比如连接不可用，网络卡顿丢包等等），资源类中的基类方法请求失败，因此返回了NULL。 可能会感觉很奇怪啊，明明我有空校验。但是，业务是复杂的，缓存的数据是从多方资源获取而来，因此，上文getDataFromResource()方法并不为空，而是有部分数据存在。

**因此导致了缓存只将部分数据写入失败**！！！！！！

### 解决方式

不要信任数据源一定是正确的，要考虑数据源可能存在不正确的方式（目前处理方式）

```
if(isempty(data)) {
    data = getDataFromResource()
    if(!isempty(data)) {
        //todo:增加数据校验
        if(isValid(data)) {
            cache.setData(data)
        }else{
            //todo:发送邮件通知，告诉开发数据可能不稳定
            mail.send();
            //todo:抛出异常，控制器处理，本次请求失败
            throw Exception();
        }
    }
}
```

或者提前计算好缓存，本次cache miss直接抛出异常，不需要计算考虑复杂的逻辑


## 案例2

案例2出现在一次上线过程中，废话不说，直接上代码。

### 代码

```
$redis = new Redis();
$redis->connect($host,$port);

$data = $redis->lrange($key, $offset, $end);
if(empty($data)) {
	$data  = getData();
	if(!empty($data)) {
	    $redis->del($key);
		$redis->rpush($key, $data);
		$redis->expire($ttl);
	}
}
```

ok,代码顺序是读=》为空的话=》读原始=》删除key,写cache。好像也没毛病吧。

![](https://oscimg.oschina.net/oscnet/805b24fa209b51cebb17a9b9b8eb547538b.jpg)

那我们放到并发场景看，来一条timeline。

![](https://oscimg.oschina.net/oscnet/dda30c0c1898ccad360b6454ba327578449.jpg)

### 解决方案

1. 利用redis事务进行处理

```
$redis->multi();
$redis->dosomething();
$redis->exex();
```
事务在并发特别高的时候回影响redis的吞吐率，但是比较可信。

2. 上锁方案

```
$res = $redis->incr(xxx);
if($res > 1) {
	//正在处理中，返回用户服务器正忙
}else{
	//写数据
}
```
加锁方案由程序员代码水平控制，比如我，就很菜。

## 案例3

案例3是个很神奇的问题

### 案例描述

crontab file:
```
$redis = new Redis();
$redis->connect($host,$port);

$data = getData();
foreach($data as $id=>$value) {
	$redis->hSet($key, $id, $value);
}
```

requeset file:
```
$redis = new Redis();
$redis->connect($host,$port);

$ids = getIds();
$value = $redis->hMget($key, $ids);
```

最后查出来的结果会出现(假设IDS如下值)：
```
array(2) {
  [2731728324]=>
  bool(false)
  [1280784723]=>
  string(16) "4290312484625424"
}
```
然后我手动通过redis命令去

	hget KEY 2731728324
	
分明是有结果的

此处一万个黑问

### 推断

在我set的过程中，只有id一直是string的，然而在我获取的过程中，id是int的，和之前的getIds()方法有关。

推断原因：

-  int 转 string hash值不一样了（经过试验，并没有）
-  int 越界（感谢[@宇润](https://my.oschina.net/u/1244455),确定为最终原因）

要进行阅读的原理代码：
[redis 字典](https://github.com/antirez/redis/blob/unstable/src/dict.c "redis 字典")

[php redis扩展](https://github.com/phpredis/phpredis/blob/develop/redis_commands.c "php redis扩展")

查看使用的php位数：

	php -i | grep System

一旦发现了x86，恭喜，你也有这样的坑。


### 测试代码 [@宇润](https://my.oschina.net/u/1244455)

```
<?php
$redis = new Redis();
$redis->connect('127.0.0.1', 6379);


$data = [
	2731728324	=>	'aaa',
	1280784723	=>	'bbb',
];

$key = 'eee';

$redis->del($key);

foreach($data as $id=>$value) {
	$redis->hSet($key, $id, $value);
}

$ids = array_keys($data);
$value = $redis->hMget($key, $ids);

var_dump($value);

foreach($ids as $id)
{
	var_dump($redis->hGet($key, $id));
}

$ids2 = [
	'2731728324',
	'1280784723',
];

$value = $redis->hMget($key, $ids2);

var_dump($value);

foreach($ids2 as $id)
{
	var_dump($redis->hGet($key, $id));
}
```

 在32位机器上 2731729324直接被转换成-1563238972

![](https://oscimg.oschina.net/oscnet/3f809416258b01a9e3e4df6ee10c417914f.jpg)

而在我读取的地方其实有另外一层封装

```
$ids = getIds();
$values = $redis->hMget($key, $ids);
$res = [];
foreach($ids as $id) {
    //这里肯定是检测不到的，在32位机器上
	if(isset($values[$id])) {
		$res[$id] =  $values[$id];
	}
}
```


### 解决

对数据写入和读取进行数据类型一致性的检测

除非特殊需求最好都统一为string,防止越界

```
写：
$redis->hSet($key, strval($id), $value);
读：
$ids = getIds();
if(is_array($ids)) {
	$ids = array_map('strval', $ids);
}
```




