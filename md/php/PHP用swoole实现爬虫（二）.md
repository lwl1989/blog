CreateTime:2017-08-19 22:07:57.0

本文大部分代码为伪代码，具体实现：[一个简单的swoole服务器](https://github.com/lwl1989/swooleFramewor)
### 如何解决worker锁住问题
按照epoll模型，master和manager只分配任务，实际执行交给worker。但是爬虫是一个超级耗时的任务，IO和CPU虽然不太损耗，
主要损耗都存在网络上（也是IO），明显受制于网络情况的变化。如（一）中所说，假设我只有2个CPU,我分配4个worker，这个时候服务器就无法处理新的请求了，这会造成资源的浪费。

所以我要寻找解决方案，首先我想到的用swoole的task解决,官方定义：

> 投递一个异步任务到task_worker池中。此函数是非阻塞的，执行完毕会立即返回。Worker进程可以继续处理新的请求。使用Task功能，必须先设置 task_worker_num，并且必须设置Server的onTask和onFinish事件回调函数。

```
int swoole_server::task(mixed $data, int $dst_worker_id = -1) 
$task_id = $serv->task("some data");
//swoole-1.8.6或更高版本
$serv->task("taskcallback", -1, function (swoole_server $serv, $task_id, $data) {
    echo "Task Callback: ";
    var_dump($task_id, $data);
});
```

### task的问题
首先我们在request事件中将任务转发到task:
```
$server->on('requset'. function(\swoole_request $request, \swoole_response $response) use($server){
    $data=json_decode($request->rawContent());
    $server->task($data);  //将任务投递到task
});
```
```
$server->on('task',function(\swoole_server $serv, $task_id, $data){
    while(true){
        //执行爬虫操作
            
        }
$server->finish();
});
```
这样的确解决的耗时异步任务的投递方式。但是因为需求会出现一个新问题，爬虫任务是一直在执行的，我们要手动使其退出，需要从服务器外部发起请求关闭task。但是，task的finish并不能关闭指定的taskId。

如何关闭taskId呢？我尝试过用swoole的sendMessage来解决：
```
$server->on('task',function($server.$taskId,$data){
    //用外部cache记录taskId
});
$server->on('request',function($req,$res){
    //假设遇到关闭信号 关闭指定TASKiD
    $server->sendMessage($data,$taskId);
});

```
但是实际上是无法关闭的，原因有二：

1. 和上面一样，while(true)不会释放控制权
2. task是一种特殊的worker_id,但是他和worker的ID一样，是从1开始，会导致系统复杂难以维护（虽然可以用$server->isTask判断）

之后我发现了一个神器:swoole_process
### swoole_process 解决方案

> swoole_process提供了如下特性：

>  1. swoole_process提供了基于unixsock的进程间通信，使用很简单只需调用write/read或者push/pop即可

>  2. swoole_process支持重定向标准输入和输出，在子进程内echo不会打印屏幕，而是写入管道，读键盘输入可以重定向为管道读取数据
>  3. 配合swoole_event模块，创建的PHP子进程可以异步的事件驱动模式
>  4. swoole_process提供了exec接口，创建的进程可以执行其他程序，与原PHP父进程之间可以方便的通信

具体实现伪代码如下:
```
$server->on('request',function($req,$res){
    if request is crawler 
        then 
            if start
                new swoole_process(function() use(data){ 
                      do crawler
                }) ;
                cache log processId
            if stop
                cache get processId
                kill processId
            if reload
                goto stop  
                goto start 
    else
        do other
});    
```
可以很好的解决上述问题，不需要维护负载的定时器，task任务等等。新增的需求是自己维护一个process map['taskname':[processId:int,stop:int]]

### 优化
前期我使用的是redis记录，会出现一定不稳定的情况(频繁读写redis，需要维护长连接等等)。
后期改用swoole_table进行维护，还可以保证任务最大数(swoole_table在使用的时候就需要初始化内存，多余的数据无法写入)，swoole table的坑：

1. 需要在server->start之前初始化
2. 需要指定详细的数据类型
3. 假如在worker中使用了create，会使其在之前的create无效并且无法在进程中共享

给个例子，避免大家走入更多的坑
```
   $table = new swoole_table(20);
    $table->column('t',swoole_table::TYPE_INT);
    $table->column('x',swoole_table::TYPE_INT);
    $table->create();
    $server = new swoole_http_server('0.0.0.0',9999);
    $server->table = $table; 

$server->on('Request', function($request, $response) use($server){
        if($request->server['request_uri'] == '/favicon.ico') {
                $response->end();
                return;
        }
$uri = ltrim($request->server['request_uri'],'/');
         $server->table->set($uri, [
                    't'     =>      1,
		    'x'     =>      1
        ]);
        $response->end('');
});
```

### 总结
由于是试验阶段，具体细节已经忘了很多，所以文中记录不太详细，有需要咨询详情的可以给我留言，也可以加我q:285753421