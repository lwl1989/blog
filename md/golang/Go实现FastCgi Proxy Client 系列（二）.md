CreateTime:2018-04-03 23:29:46.0

如果没有看前篇，请点击[Go实现FastCgi Proxy Client 系列（一）](https://my.oschina.net/lwl1989/blog/1788957)，文内篇一，都是指这个

# http header的构建

### golang的http包

golang的http包网上demo实在是太多了，我就不copy了，从golang的core/src来看吧。

net/http

首先进入到golang的安装目录，可以看到src目录，进入到其中，可以找到net/http目录。

![内核中的net/http](https://static.oschina.net/uploads/img/201804/03223611_Csci.jpg "在这里输入图片标题")

可以发现，这里几乎涵盖了http所有要用的功能。

那我们要讲请求转发，自然就要看到request.go。

### golang http的request
golang 对Request对象（姑且以面向对象的方式解释）定义如下:
```
type Request struct {                       //忽略这mardown语法影响显示}
	Method           string
	URL              *url.URL
	Proto            string
	ProtoMajor       int
	ProtoMinor       int    
	Header           Header
	Body             io.ReadCloser
	ContentLength    int64
	TransferEncoding []string
	Close            bool
	Host             string
	Form             url.Values
	PostForm         url.Values
	MultipartForm    *multipart.Form
	Trailer          Header
	RemoteAddr       string
	RequestURI       string
	TLS              *tls.ConnectionState
	Cancel <-chan    struct{}
	Response         *Response
	ctx              context.Context
    }
```
那我们就可以找到大部分我们要的东西。
1. 请求header头，我们可以从Header中获取
2. 请求体，我们可以从Body中获取，如果是表单，我们就要从Form、PostForm或者MultipartForm获取
3. 其他信息，比如host，url之类，我们就可以完美的获取到要的信息。

按照[篇一](https://my.oschina.net/lwl1989/blog/1788957)所讲，我们要生成的Header,可以在这里获取，当然，不是全部。比如document_root这种，[具体的代码实现在这](https://github.com/lwl1989/spinx/blob/master/core/request.go)。

### response
 
response这里就不详细介绍了，主要看我们如何接收数据吧，功能主要是讲接收到的数据返回给用户。

### 接收数据
   
[篇一](https://my.oschina.net/lwl1989/blog/1788957)稍微贴了一个代码，为了更直观的体现，我这里再贴一次。

```
    rec := &record{} //建立一个记录缓冲
	var err1 error

	// recive untill EOF or FCGI_END_REQUEST
	for {
		err1 = rec.read(cgi.rwc)   //从cgi建立的链接读取数据
		if err1 != nil {  //判断是否有错误，判断错误是否为终止符
			if err1 != io.EOF {  // keepalive on的时候 不会有终止符  这个要注意
				err = err1
			}
			break
		}
		switch {  //根据返回的类型，将内容写到响应的字节数组中
		case rec.h.Type == typeStdout:   
			retout = append(retout, rec.content()...)
		case rec.h.Type == typeStderr:
			reterr = append(reterr, rec.content()...)
		case rec.h.Type == typeEndRequest:
			fallthrough
		default:
			break
		}
	}
```
仔细观察type，就是我们在[篇一](https://my.oschina.net/lwl1989/blog/1788957)说的消息类型的几种。

我们在假设没有错误出现的情况下，retout这个数组自然是获取所有的返回值。这个时候，golang存储的是字节[22 35 97 ......]，直接将其转化成字符串就可以看到我们熟悉的Http协议的Response了。

如：
```
头部：
Cache-Control: no-cache, must-revalidate, max-age=0
Connection: keep-alive
Content-Type: text/html; charset=utf-8
Date: Tue, 03 Apr 2018 15:10:58 GMT
Expires: Wed, 11 Jan 1984 05:00:00 GMT
Server: nginx/1.12.2
Transfer-Encoding: chunked
X-Powered-By: PHP/7.1.9

正文(记住正文上面有2个换行符):
//......
```
这个时候，我们只需要将我们获取到的内容格式化我们想要的内容即可。

代码太多，我就不贴了，给个链接 [spinx http Response的实现](https://github.com/lwl1989/spinx/blob/master/core/response.go)。

# 坑的记录 

### content-length

我实现好之后，用fpm做试验，测试post的时候，发现如论如何php都接收不到post的值。（包括$_POST和php://input）

但是我用官方实现的 net/http/fcgi 做了一个demo又能收到数据，迷糊了半天。

后来在swoole群里获得了群友 大白菜、夕阳下的奔跑、Neo  的帮助，发现是PHP有必须对http的content-length进行读，得到长度之后，才会对 body的包进行读取。

所以，我们要加上 header map对content-length的设置

### cookies无法设置成功

这个位置我重写了我的赋值位置，不能直接使用获取的 Set-Cookies 的Header进行处理，需要将重写成Cookies，并且 http.SetCookie(write,cookie) 用内置的方法输出。

# 性能测试

几乎和nginx没区别，证明我fpm进程数开得不够多，我下次多开点fpm再来进行压测（以wordpress进行的基准测试）。
```
new version test(wordpress):

Server Software:        spinx
Server Hostname:        www.test.com
Server Port:            18000

Document Path:          /
Document Length:        53302 bytes

Concurrency Level:      100
Time taken for tests:   20.158 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      53507000 bytes
HTML transferred:       53302000 bytes
Requests per second:    49.61 [#/sec] (mean)
Time per request:       2015.799 [ms] (mean)
Time per request:       20.158 [ms] (mean, across all concurrent requests)
Transfer rate:          2592.17 [Kbytes/sec] received
==========================================================
Server Software:        nginx/1.13.9
Server Hostname:        www.test.com
Server Port:            18000

Document Path:          /
Document Length:        53302 bytes

Concurrency Level:      100
Time taken for tests:   20.277 seconds
Complete requests:      1000
Failed requests:        0
Total transferred:      53548000 bytes
HTML transferred:       53302000 bytes
Requests per second:    49.32 [#/sec] (mean)
Time per request:       2027.685 [ms] (mean)
Time per request:       20.277 [ms] (mean, across all concurrent requests)
Transfer rate:          2578.95 [Kbytes/sec] received
=============================================================
Performance has caught up with nginx.
```

# spinx的github

给力点，大哥们，来点star吧。

[spinx 一个golang实现的fastcgi proxy client](https://github.com/lwl1989/spinx)