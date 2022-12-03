CreateTime:2016-07-04 18:29:52.0

##预备知识
###Generator
```
function my_range($start, $end, $step = 1) {  
    for ($i = $start; $i <= $end; $i += $step) {  
        yield $i;  
    }  
}  
  
foreach (my_range(1, 1000) as $num) {  
    echo $num, "\n";  
}  
/* 
 * 1 
 * 2 
 * ... 
 * 1000 
 */  

$range = my_range(1, 1000);  
  
var_dump($range);  
/* 
 * object(Generator)#1 (0) { 
 * } 
 */  
  
var_dump($range instanceof Iterator);  
/* 
 * bool(true) 
 */  
```
由于接触PHP时日尚浅，并未深入语言实现细节，所以只能根据现象进行猜测，以下是我的一些个人理解：
包含yield关键字的函数比较特殊，返回值是一个Generator对象，此时函数内语句尚未真正执行
Generator对象是Iterator接口实例，可以通过rewind()、current()、next()、valid()系列接口进行操纵
Generator可以视为一种“可中断”的函数，而yield构成了一系列的“中断点”
Generator类似于车间生产的流水线，每次需要用产品的时候才从那里取一个，然后这个流水线就停在那里等待下一次取操作
###Coroutine
细心的读者可能已经发现，截至目前，其实Generator已经实现了Coroutine的关键特性：中断执行、恢复执行。按照《当C/C++后台开发遇上Coroutine》的思路，借助“全局变量”一类语言设施进行信息传递，实现异步Server应该足够了。
其实相对于swapcontext族函数，Generator已经前进了一大步，具备了“返回数据”的能力，如果同时具备“发送数据”的能力，就再也不必通过那些蹩脚的手法绕路而行了。在PHP里面，通过Generator的send()接口（注意：不再是next()接口），可以完成“发送数据”的任务，从而实现了真正的“双向通信”。
```
function gen() {  
    $ret = (yield 'yield1');  
    echo "[gen]", $ret, "\n";  
    $ret = (yield 'yield2');  
    echo "[gen]", $ret, "\n";  
}  
  
$gen = gen();  
$ret = $gen->current();  
echo "[main]", $ret, "\n";  
$ret = $gen->send("send1");  
echo "[main]", $ret, "\n";  
$ret = $gen->send("send2");  
echo "[main]", $ret, "\n";  
  
/* 
 * [main]yield1 
 * [gen]send1 
 * [main]yield2 
 * [gen]send2 
 * [main] 
 */  
```
作为C/C++系码农，发现“可重入”、“双向通信”能力之后，貌似没有更多奢求了，不过PHP还是比较慷慨，继续添加了Exception机制，“错误处理”机制得到进一步完善。

```
function gen() {  
    $ret = (yield 'yield1');  
    echo "[gen]", $ret, "\n";  
    try {  
        $ret = (yield 'yield2');  
        echo "[gen]", $ret, "\n";  
    } catch (Exception $ex) {  
        echo "[gen][Exception]", $ex->getMessage(), "\n";  
    }     
    echo "[gen]finish\n";  
}  
  
$gen = gen();  
$ret = $gen->current();  
echo "[main]", $ret, "\n";  
$ret = $gen->send("send1");  
echo "[main]", $ret, "\n";  
$ret = $gen->throw(new Exception("Test"));  
echo "[main]", $ret, "\n";  
  
/* 
 * [main]yield1 
 * [gen]send1 
 * [main]yield2 
 * [gen][Exception]Test 
 * [gen]finish 
 * [main] 
 */  
```
##实战演习
前面简单介绍了相关的语言设施，那么具体到实际项目中，到底应该如何运用呢？让我们继续《一次失败的PHP扩展开发之旅》描述的场景，借助上述特性实现那个美好的愿望：以同步方式书写异步代码！
###第一版初稿
```
<?php  
  
class AsyncServer {  
    protected $handler;  
    protected $socket;  
    protected $tasks = [];  
  
    public function __construct($handler) {  
        $this->handler = $handler;  
  
        $this->socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);  
        if(!$this->socket) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
        if (!socket_set_nonblock($this->socket)) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
        if(!socket_bind($this->socket, "0.0.0.0", 1234)) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
    }  
  
    public function Run() {  
        while (true) {  
            $reads = array($this->socket);  
            foreach ($this->tasks as list($socket)) {  
                $reads[] = $socket;  
            }  
            $writes = NULL;  
            $excepts= NULL;  
            if (!socket_select($reads, $writes, $excepts, 0, 1000)) {  
                continue;  
            }  
  
            foreach ($reads as $one) {  
                $len = socket_recvfrom($one, $data, 65535, 0, $ip, $port);  
                if (!$len) {  
                    //echo "socket_recvfrom fail.\n";  
                    continue;  
                }  
                if ($one == $this->socket) {  
                    //echo "[Run]request recvfrom succ. data=$data ip=$ip port=$port\n";  
                    $handler = $this->handler;  
                    $coroutine = $handler($one, $data, $len, $ip, $port);  
                    $task = $coroutine->current();  
                    //echo "[Run]AsyncTask recv. data=$task->data ip=$task->ip port=$task->port timeout=$task->timeout\n";  
                    $socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);  
                    if(!$socket) {  
                        //echo socket_strerror(socket_last_error())."\n";  
                        $coroutine->throw(new Exception(socket_strerror(socket_last_error()), socket_last_error()));  
                        continue;  
                    }  
                    if (!socket_set_nonblock($socket)) {  
                        //echo socket_strerror(socket_last_error())."\n";  
                        $coroutine->throw(new Exception(socket_strerror(socket_last_error()), socket_last_error()));  
                        continue;  
                    }  
                    socket_sendto($socket, $task->data, $task->len, 0, $task->ip, $task->port);  
                    $this->tasks[$socket] = [$socket, $coroutine];  
                } else {  
                    //echo "[Run]response recvfrom succ. data=$data ip=$ip port=$port\n";  
                    if (!isset($this->tasks[$one])) {  
                        //echo "no async_task found.\n";  
                    } else {  
                        list($socket, $coroutine) = $this->tasks[$one];  
                        unset($this->tasks[$one]);  
                        socket_close($socket);  
                        $coroutine->send(array($data, $len));  
                    }  
                }  
            }  
        }  
    }  
}  
  
class AsyncTask {  
    public $data;  
    public $len;  
    public $ip;  
    public $port;  
    public $timeout;  
  
    public function __construct($data, $len, $ip, $port, $timeout) {  
        $this->data = $data;  
        $this->len = $len;  
        $this->ip = $ip;  
        $this->port = $port;  
        $this->timeout = $timeout;  
    }  
}  
  
function RequestHandler($socket, $req_buf, $req_len, $ip, $port) {  
    //echo "[RequestHandler] before yield AsyncTask. REQ=$req_buf\n";  
    list($rsp_buf, $rsp_len) = (yield new AsyncTask($req_buf, $req_len, "127.0.0.1", 2345, 1000));  
    //echo "[RequestHandler] after yield AsyncTask. RSP=$rsp_buf\n";  
    socket_sendto($socket, $rsp_buf, $rsp_len, 0, $ip, $port);  
}  
  
$server = new AsyncServer(RequestHandler);  
$server->Run();  
  
?>  
```
代码解读：

为了便于说明问题，这里所有底层通讯基于UDP，省略了TCP的connect等繁琐细节
AsyncServer为底层框架类，封装了网络通讯细节以及协程切换细节，通过socket进行coroutine绑定
RequestHandler为业务处理函数，通过yield new AsyncTask()实现异步网络交互
###第二版完善
第一版遗留问题：

异步网络交互的timeout未实现，仅预留了接口参数
yield new AsyncTask()调用方式不够自然，略感别扭
```
<?php  
  
class AsyncServer {  
    protected $handler;  
    protected $socket;  
    protected $tasks = [];  
    protected $timers = [];  
  
    public function __construct(callable $handler) {  
        $this->handler = $handler;  
  
        $this->socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);  
        if(!$this->socket) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
        if (!socket_set_nonblock($this->socket)) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
        if(!socket_bind($this->socket, "0.0.0.0", 1234)) {  
            die(socket_strerror(socket_last_error())."\n");  
        }  
    }  
  
    public function Run() {  
        while (true) {  
            $now = microtime(true) * 1000;  
            foreach ($this->timers as $time => $sockets) {  
                if ($time > $now) break;  
                foreach ($sockets as $one) {  
                    list($socket, $coroutine) = $this->tasks[$one];  
                    unset($this->tasks[$one]);  
                    socket_close($socket);  
                    $coroutine->throw(new Exception("Timeout"));  
                }  
                unset($this->timers[$time]);  
            }  
  
            $reads = array($this->socket);  
            foreach ($this->tasks as list($socket)) {  
                $reads[] = $socket;  
            }  
            $writes = NULL;  
            $excepts= NULL;  
            if (!socket_select($reads, $writes, $excepts, 0, 1000)) {  
                continue;  
            }  
  
            foreach ($reads as $one) {  
                $len = socket_recvfrom($one, $data, 65535, 0, $ip, $port);  
                if (!$len) {  
                    //echo "socket_recvfrom fail.\n";  
                    continue;  
                }  
                if ($one == $this->socket) {  
                    //echo "[Run]request recvfrom succ. data=$data ip=$ip port=$port\n";  
                    $handler = $this->handler;  
                    $coroutine = $handler($one, $data, $len, $ip, $port);  
                    if (!$coroutine) {  
                        //echo "[Run]everything is done.\n";  
                        continue;  
                    }  
                    $task = $coroutine->current();  
                    //echo "[Run]AsyncTask recv. data=$task->data ip=$task->ip port=$task->port timeout=$task->timeout\n";  
                    $socket = socket_create(AF_INET, SOCK_DGRAM, SOL_UDP);  
                    if(!$socket) {  
                        //echo socket_strerror(socket_last_error())."\n";  
                        $coroutine->throw(new Exception(socket_strerror(socket_last_error()), socket_last_error()));  
                        continue;  
                    }  
                    if (!socket_set_nonblock($socket)) {  
                        //echo socket_strerror(socket_last_error())."\n";  
                        $coroutine->throw(new Exception(socket_strerror(socket_last_error()), socket_last_error()));  
                        continue;  
                    }  
                    socket_sendto($socket, $task->data, $task->len, 0, $task->ip, $task->port);  
                    $deadline = $now + $task->timeout;  
                    $this->tasks[$socket] = [$socket, $coroutine, $deadline];  
                    $this->timers[$deadline][$socket] = $socket;  
                } else {  
                    //echo "[Run]response recvfrom succ. data=$data ip=$ip port=$port\n";  
                    list($socket, $coroutine, $deadline) = $this->tasks[$one];  
                    unset($this->tasks[$one]);  
                    unset($this->timers[$deadline][$one]);  
                    socket_close($socket);  
                    $coroutine->send(array($data, $len));  
                }  
            }  
        }  
    }  
}  
  
class AsyncTask {  
    public $data;  
    public $len;  
    public $ip;  
    public $port;  
    public $timeout;  
  
    public function __construct($data, $len, $ip, $port, $timeout) {  
        $this->data = $data;  
        $this->len = $len;  
        $this->ip = $ip;  
        $this->port = $port;  
        $this->timeout = $timeout;  
    }  
}  
  
function AsyncSendRecv($req_buf, $req_len, $ip, $port, $timeout) {  
    return new AsyncTask($req_buf, $req_len, $ip, $port, $timeout);  
}  
  
function RequestHandler($socket, $req_buf, $req_len, $ip, $port) {  
    //echo "[RequestHandler] before yield AsyncTask. REQ=$req_buf\n";  
    try {  
        list($rsp_buf, $rsp_len) = (yield AsyncSendRecv($req_buf, $req_len, "127.0.0.1", 2345, 3000));  
    } catch (Exception $ex) {  
        $rsp_buf = $ex->getMessage();  
        $rsp_len = strlen($rsp_buf);  
        //echo "[Exception]$rsp_buf\n";  
    }  
    //echo "[RequestHandler] after yield AsyncTask. RSP=$rsp_buf\n";  
    socket_sendto($socket, $rsp_buf, $rsp_len, 0, $ip, $port);  
}  
  
$server = new AsyncServer(RequestHandler);  
$server->Run();  
  
?>  
```
代码解读：

借助PHP内置array能力，实现简单的“超时管理”，以毫秒为精度作为时间分片
封装AsyncSendRecv接口，调用形如yield AsyncSendRecv()，更加自然
添加Exception作为错误处理机制，添加ret_code亦可，仅为展示之用
##性能测试

![输入图片说明](http://img.blog.csdn.net/20141124161616626 "在这里输入图片标题")

100Byte/REQ	        1000Byte/REQ
async_svr_v1.php	16000/s	15000/s
async_svr_v2.php	11000/s	10000/s
##展望未来
有兴趣的PHPer可以基于该思路进行底层框架封装，对于常见阻塞操作进行封装，比如：connect、send、recv、sleep ...
本人接触PHP时日尚浅，很多用法非最优，高手可有针对性优化，性能应该可以继续提高
目前基于socket进行coroutine绑定，如果基于TCP通信，每次connect/close，开销过大，需要考虑实现连接池
python等语言也有类似的语言设施，有兴趣的读者可以自行研究