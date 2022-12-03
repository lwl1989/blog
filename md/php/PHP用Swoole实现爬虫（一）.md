CreateTime:2017-08-12 22:53:40.0

### 基本概念

##### 网络爬虫

> 网络爬虫（又被称为网页蜘蛛，网络机器人，在FOAF社区中间，更经常的称为网页追逐者），是一种按照一定的规则，自动地抓取万维网信息的程序或者脚本。另外一些不常使用的名字还有蚂蚁、自动索引、模拟程序或者蠕虫。

##### swoole


> PHP的异步、并行、高性能网络通信引擎，使用纯C语言编写，提供了PHP语言的异步多线程服务器，异步TCP/UDP网络客户端，异步MySQL，异步Redis，数据库连接池，AsyncTask，消息队列，毫秒定时器，异步文件读写，异步DNS查询。 Swoole内置了Http/WebSocket服务器端/客户端、Http2.0服务器端。

### 技术方案

本来，公司的意愿是，写几个PHP脚本，使用linux定时任务crontab既可。后来我一琢磨，正好现在不是业务改动频繁期，而且，服务化是迟早要做的事情，因此，便开始我的爬坑旅程。
PHP是有一套比较成熟的异步常驻内存的框架的，[ **workerman** ](http://www.workerman.net/),倒不是不采取，而是既然决定采用新的方案，正好也不赶工期，为何不挑战一下新技术呢？

### 爬坑之旅（一）

##### swoole协议选取

由于公司之前是没有进行过tcp连接的优化基础的大神在，因此我们采用是朴实的方案，http协议。
swoole是原生支持http服务器的，参见官网，开启一个http server是很简单的：
```
$serv = new Swoole\Http\Server("127.0.0.1", 9502);

$serv->on('Request', function($request, $response) {
    var_dump($request->get);
    var_dump($request->post);
    var_dump($request->cookie);
    var_dump($request->files);
    var_dump($request->header);
    var_dump($request->server);

    $response->cookie("User", "Swoole");
    $response->header("X-Server", "Swoole");
    $response->end("<h1>Hello Swoole!</h1>");
});

$serv->start();
```
根据swoole官方的定义，http server是有几个常用事件的。
```
request,packet,pipeMessage,task,finish,receive,close,workerStart,workerStop,shutDown
```
仔细观察，一般的使用的情况下，由于receive事件swoole已经自动转发到了request事件，因此，最简单的例子，我们直接从request中激活一个爬虫就好了。

##### 初始的代码

```
class SwooleHttpServer implements Server
{
        const EVENT = [
                'request'//,'packet','pipeMessage','task','finish','close'
        ];
        protected $server;
        protected $event = [

        ];
 //注意，这里使用了我上一篇文章关于PHP DI的实现
        public function __construct(Config $config)   
        {
                $server = $config->get('server');
                if(empty($server)) {
                        throw new \Exception('config not found');
                }
                $this->server = new \swoole_http_server($server['host'], $server['port'], $server['mode'], $server['type']);

                $extend = $config->get('event')['namespace'] ?? '';
                foreach (self::EVENT as $event) {
                        $class = $extend.'\\'.ucfirst($event);

                        if(!class_exists($class)) {
                                $class = '\\Kernel\\Swoole\\Event\\Http\\'.ucfirst($event);
                        }
                        /* @var \Kernel\Swoole\Event\Event $callback */
                        $callback = new $class($this->server);
                        $this->event[$event] = $callback;
                        $this->server->on($event, [$callback, 'doEvent']);
                }
                $this->server->set($config->get('swoole'));
        }

        public function start(\Closure $callback = null): Server
        {
                if(!is_null($callback)) {
                        $callback();
                }
                $this->server->start();
                return $this;
        }

        public function shutdown(\Closure $callback = null): Server
        {
                // TODO: Implement shutdown() method.
        }

        public function close($fd, $fromId = 0) : Server
        {
                $this->server->close($fd, $fromId = 0);
                return $this;
        }
```
为了方便业务的书写，我们把request事件解耦出单独的一个类
```
namespace Kernel\Swoole\Event\Http;

use Kernel\Swoole\Event\Event;
use Kernel\Swoole\Event\EventTrait;

class Request implements Event
{
        use EventTrait;
        /* @var  \swoole_http_server $server*/

        protected $server;
        protected $data = [];

        public function __construct(\swoole_http_server $server)
        {
                $this->server = $server;
        }
       public function doEvent(\swoole_http_request $request, \swoole_http_response $response)
        {
                if(isset($request->server['request_uri']) and $request->server['request_uri'] == '/favicon.ico') {
                        $response->end(json_encode(empty($data)?['code'=>0]:$data));
                        return;
                }
                $crawler = new Crawler();
               $crawler->run();
        }
```
现在我们请求IP所在端口，既可开始执行我们得爬虫类了

##### 最初最简单的爬虫类

```
namespace Library\Crawler;


class Crawler
{
        private $url;
        private $toVisit = [];

        public function __construct($url)
        {
                $this->url = $url;
        }

        public function visitOneDegree()
        {
                $this->visit($this->url, function ($content) {
                        $this->loadPage($content);
                        $this->visitAll();
                });
        }


        private function loadPage($content)
        {
                $pattern = '#(http|ftp|https)://?([a-z0-9_-]+\.)+(com|net|cn|org){1}(\/[a-z0-9_-]+)*\.?(?!:jpg|jpeg|gif|png|bmp)(?:")#i';
                preg_match_all($pattern, $content, $matched);
                foreach ($matched[0] as $url) {
                        if (in_array($url, $this->toVisit)) {
                                continue;
                        }
                        $this->toVisit[] = rtrim($url,'"');
                        file_put_contents('urls',$url."\
",FILE_APPEND);
                }
        }

        private function visitAll()
        {
                foreach ($this->toVisit as $url) {
                        $this->visit($url);
                }
        }

        private function visit($url, $callback = null)
        {
                $urlInfo = parse_url($url);
                \Swoole\Async::dnsLookup($urlInfo['host'], function ($domainName, $ip) use($urlInfo,$url,$callback) {
                        if($domainName == '' or $ip =='') {
                                return;
                        }
                        if(!isset($urlInfo['port'])) {
                                if($urlInfo['scheme'] == 'https') {
                                        $urlInfo['port'] = 443;
                                }else{
                                        $urlInfo['port'] = 80;
                                }
                        }
                        if($urlInfo['scheme'] == 'https') {
                                $cli = new \swoole_http_client($ip,  $urlInfo['port'], true);
                        }else{
                                $cli = new \swoole_http_client($ip,  $urlInfo['port']);
                        }

                        $cli->setHeaders([
                                'Host' => $domainName,
                                "User-Agent" => 'Chrome/49.0.2587.3',
                                'Accept' => 'text/html,application/xhtml+xml,application/xml',
                                'Accept-Encoding' => 'gzip',
                        ]);
                        $cli->get($urlInfo['path']??'/', function ($cli) use (,$url) {
                                $data = $this->getMeta($cli->body);
                                //todo:将数据写到数据库
                                $cli->close();
                        });

                });

        }


        private function getMeta(string $data)
        {
                $meta = [];
               ......
                return $meta;
        }
}

```
从现在开始，一套简单的爬虫程序既可使用了。
但是出现了一个问题：
    
swoole的worker数量受制于CPU有限，因此，一旦超出了worker是不会进行服务的，而我这里的爬虫，很明显是一个同步代码,举个栗子，开始我worker_num是5，我就最多同时开启5个爬虫任务，就算你想立即开启更多，也是会失败的【swoole分配不了更多的worker进程给请求】。相当于
```
while(true){
    if($condition){
        break;
        }
}
```
引用一张官方的进程图：

![输入图片说明](https://wiki.swoole.com/static/image/process.jpg "在这里输入图片标题")

worker的数量是不能动态分配的。联系操作系统的知识，可以得到这样的结论，假如我有3个篮子（即worker[生产者]），每次有人来取走一个篮子去装东西（即request请求[消费者]），但是，篮子一直在被用着(while(true))没还回来。当有第4个人来拿篮子（请求）的时候，无法分配篮子，只能等待，假如之前的一直不释放，while(true)不退出，这就称之为死锁了。


因此，我们需要进行新一步的优化。
下一篇文章，我将讲述如何对swoole的进程进行优化，也就是最大的爬坑篇[swoole task]。

### 参考

[swoole官方文档](https://wiki.swoole.com/wiki/index/prid-1)