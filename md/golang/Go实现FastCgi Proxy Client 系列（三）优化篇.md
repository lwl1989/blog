CreateTime:2018-05-15 19:29:42.0

# 墨迹一点

#### 个人琐碎
最近比较忙，以致于很久都没有写blog了，但是，golang的水平自认为是总算入门了。

#### 协程的个人理解

网上的说法一般都是协程是轻量级线程。

我个人认为协程的好处

1. 小
2. 无需在用户态和内核态切换（完全在用户态）
3. 无需线程上下文切换的开销（因为之上的好处）
4.  编码简单（原子操作，锁都没有了）
    

# httpHandler优化
    
#### 利用协程优化请求
  
这是我们原本的handler（就是监听http请求的一个对象，实现了ServeHTTP）

```
type HttpHandler struct
{   
    Vhosts     Vhosts
    HandlerMap map[string]*http.ServeMux
}
```
很明显，这里就已经是入口了，我们将其修改成

```
type HttpHandler struct
{   
    Vhosts            Vhosts
    HandlerMap        map[string]*http.ServeMux
    Response          chan *Response
    StaticFile        chan *StaticFileHandler
    serverEnvironment map[string]string
}
```

当我们遇到请求的实行，我们直接开启协程
```
go Run(w,r)
```
在Run方法中，将实现部分写入协程(即IO、计算部分),在大部分代码不变的情况下（请看1和2的分析）我们将几种错误的情况抛出一个Response对象，比如请求的是/favicon.ico 
```
if r.RequestURI == "/favicon.ico" {
		httpHandler.Response <- &Response{200, map[string]string{}, nil, ""}
		return
}
```
具体的伪代码应如下：
```
if something wrong { 
    gerWrongCode 
    res := generate WrongResponse
    httpHandler.Response <- res
    return 
}


res := do proxy
httpHandler.Response <- res
```
另外，关于静态文件，我们并不需要proxy，所以我们要告知上面，这个是静态文件，go直接处理
```
        fileCode,filename := httpHandler.buildServerHttp(r, env, hm)

		switch fileCode {
		case FileCodeStatic:
			httpHandler.StaticFile <- &StaticFileHandler{
				name,
				port,
				filename,
			}
			return
            case ......
        }
```
#### 处理协程回调
那么确定我们已经将各种IO和逻辑处理写入了协程了，这个时候，我们回到ServeHTTP方法
```
    go Run(w,r)
    for {
		select {    //进行协程的处理
		case response := <-httpHandler.Response:  //当遇到response的时候 送出结果
			response.send(w, r)
		case hand := <-httpHandler.StaticFile:    //当遇到是静态文件的时候 直接走本身go原本的handler
			staticHandler := httpHandler.HandlerMap[hand.Host+hand.Port]
			staticHandler.ServeHTTP(w, r)

		default:
			respond(w, "<h1>404</h1>", 404, map[string]string{})
		}
	}
```

# 添加日志
像nginx之类的，都可以写日志，那这个功能我也不能少

只需要在HttpHandler对象注入的时候加入一个log即可
```
type HttpHandler struct
{   
	Vhosts            Vhosts
	HandlerMap        map[string]*http.ServeMux
	Response          chan *Response
	StaticFile        chan *StaticFileHandler
	serverEnvironment map[string]string
	log               *log.Logger
}

func (httpHandler *HttpHandler) SetLogger(log *log.Logger) {
	httpHandler.log = log
}

func (httpHandler *HttpHandler)  GetLogger() *log.Logger {
	return httpHandler.log
}
```

然后，代码中可以放心大胆的使用
log.*方法

至于log写文件 ，百度谷歌谢谢，他本身就带了异步IO，就不用操心了。

# 其他

[本文源码](https://github.com/lwl1989/spinx)

[Go实现FastCgi Proxy Client 系列（一）](https://my.oschina.net/lwl1989/blog/1788957)

[Go实现FastCgi Proxy Client 系列（二）](https://my.oschina.net/lwl1989/blog/1789583)

[一个FCM的消息代理服务器](https://github.com/lwl1989/TTTask)

[一个秒级定时任务（非crontab）](https://github.com/lwl1989/timing)