CreateTime:2017-03-03 14:17:59.0

#测试代码
```
$msg = ['test'=>23];
$start = microtime(true);  
for($i=0;$i<100000;$i++){
	$packMsg = msgpack_pack($msg);
}
echo 'pack len:'.strlen($packMsg)."\
";
$end = microtime(true);
echo 'run time:'.($end-$start).'s'."\
";  
echo 'memory usage:'.(memory_get_usage()/1024)."KB\
";
/*
$start = microtime(true);  
for($i=0;$i<100000;$i++){
	$jsonMsg = json_encode($msg);
}
echo 'json len:'.strlen($jsonMsg)."\
";
$end = microtime(true);  
echo 'run time:'.($end-$start).'s'."\
";  
echo 'memory usage:'.(memory_get_usage()/1024)."KB\
";

$start = microtime(true);  
for($i=0;$i<100000;$i++){
	$packMsg = serialize($msg);
}
echo 'php len:'.strlen($packMsg)."\
";
$end = microtime(true);
echo 'run time:'.($end-$start)."s\
";
echo 'memory usage:'.(memory_get_usage()/1024)."KB\
";*/


```
#执行结果
```
pack len:7
run time:0.024219989776611s
memory usage:354.4765625KB
json len:11
run time:0.010890007019043s
memory usage:354.1796875KB
php len:22
run time:0.010586977005005s
memory usage:353.8828125KB
```
#分析评论
网上查阅的基本结果都是(估计是php7以前的版本)

    运行速度  serialize<json<msgpack
    长度   serialize>json>msgpack
    内存消耗  serialize<json<msgpack  //不过近乎一致

在php7里运行，得出的结果如下
    
    运行速度  serialize<msgpack<json   //这里出现了变化
    长度   serialize>json>msgpack
    内存消耗  serialize<json<msgpack  //不过近乎一致


    