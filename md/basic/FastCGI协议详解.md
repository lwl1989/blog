CreateTime:2019-08-07 17:57:43.0

[麻省理工-协议文档](http://www.mit.edu/~yandros/doc/specs/fcgi-spec.html)

[go的转发器实现](https://github.com/lwl1989/spinx)

[Go实现FastCgi Proxy Client 系列（四）](https://my.oschina.net/lwl1989/blog/1823443)

[Go实现FastCgi Proxy Client 系列（三）](https://my.oschina.net/lwl1989/blog/1813102)

[Go实现FastCgi Proxy Client 系列（二）](https://my.oschina.net/lwl1989/blog/1789583)

[Go实现FastCgi Proxy Client 系列（一）](https://my.oschina.net/lwl1989/blog/1788957)

# cgi

### CGI

CGI全称是“公共网关接口”(Common Gateway Interface)，HTTP服务器与你的或其它机器上的程序进行“交谈”的一种工具，其程序须运行在网络服务器上。

#### cgi的弊端

cgi会产生什么问题呢？

当每一个请求进入的时候，cgi都会fork一个新的进程，然后以php为例，每个请求都要耗费相当大的内存，这样一来，并发起来，完全就会GG。

为了解决这个问题，于是产生了fastCgi。

### FastCGI

FastCGI像是一个常驻(long-live)型的CGI，它可以一直执行着，只要激活后，不会每次都要花费时间去fork一次（这是CGI最为人诟病的fork-and-execute 模式）。

它还支持分布式的运算，即 FastCGI 程序可以在网站服务器以外的主机上执行并且接受来自其它网站服务器来的请求。

# 一次完整的http经cgi请求到后端的分析

http request:
```
Request URL: http://demo.inke.cn/server.php
Request Method: GET
Status Code: 200 OK
Remote Address: 127.0.0.1:80
Referrer Policy: no-referrer-when-downgrade
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cache-Control: max-age=0
Connection: keep-alive
Cookie: aid=fa29caf3-5b18-401c-8abd-677eb1a1c45f; s-59804d897bc03a0036dc141c=2a126bda133d4feba4169fa70308f2c7
Host: demo.inke.cn
Upgrade-Insecure-Requests: 1
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36
```

server.php
```
<?php
 
 
var_dump($_SERVER);
返回结果：
array(33) {
  ["USER"]=>
  string(6) "limars"
  ["HOME"]=>
  string(13) "/Users/limars"
  ["HTTP_COOKIE"]=>
  string(101) "aid=d11e3484-fd7d-4c6e-a2a5-bfabe2271da2; s-59804d897bc03a0036dc141c=fcb6a8cf23644f50b66e405d173eced7"
  ["HTTP_ACCEPT_LANGUAGE"]=>
  string(14) "zh-CN,zh;q=0.9"
  ["HTTP_ACCEPT_ENCODING"]=>
  string(13) "gzip, deflate"
  ["HTTP_ACCEPT"]=>
  string(118) "text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3"
  ["HTTP_USER_AGENT"]=>
  string(121) "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_13_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/75.0.3770.142 Safari/537.36"
  ["HTTP_UPGRADE_INSECURE_REQUESTS"]=>
  string(1) "1"
  ["HTTP_CONNECTION"]=>
  string(10) "keep-alive"
  ["HTTP_HOST"]=>
  string(12) "demo.inke.cn"
  ["REDIRECT_STATUS"]=>
  string(3) "200"
  ["SERVER_NAME"]=>
  string(12) "demo.inke.cn"
  ["SERVER_PORT"]=>
  string(2) "80"
  ["SERVER_ADDR"]=>
  string(9) "127.0.0.1"
  ["REMOTE_PORT"]=>
  string(5) "51429"
  ["REMOTE_ADDR"]=>
  string(9) "127.0.0.1"
  ["SERVER_SOFTWARE"]=>
  string(12) "nginx/1.12.2"
  ["GATEWAY_INTERFACE"]=>
  string(7) "CGI/1.1"
  ["REQUEST_SCHEME"]=>
  string(4) "http"
  ["SERVER_PROTOCOL"]=>
  string(8) "HTTP/1.1"
  ["DOCUMENT_ROOT"]=>
  string(31) "/Users/limars/Desktop/inke/demo"
  ["DOCUMENT_URI"]=>
  string(11) "/server.php"
  ["REQUEST_URI"]=>
  string(11) "/server.php"
  ["SCRIPT_NAME"]=>
  string(11) "/server.php"
  ["CONTENT_LENGTH"]=>
  string(0) ""
  ["CONTENT_TYPE"]=>
  string(0) ""
  ["REQUEST_METHOD"]=>
  string(3) "GET"
  ["QUERY_STRING"]=>
  string(0) ""
  ["SCRIPT_FILENAME"]=>
  string(42) "/Users/limars/Desktop/inke/demo/server.php"
  ["FCGI_ROLE"]=>
  string(9) "RESPONDER"
  ["PHP_SELF"]=>
  string(11) "/server.php"
  ["REQUEST_TIME_FLOAT"]=>
  float(1565082651.4024)
  ["REQUEST_TIME"]=>
  int(1565082651)
}
```

它是个怎样的转换过程呢？仔细观察，我们会发现有很多东西是http协议里没有的，那这些东西是怎么生成的呢？

观察nginx配置：
```
listen       80;
server_name  demo.inke.cn;
root        /Users/limars/Desktop/inke/demo;
 
 location ~ \.php$ {
            try_files        $uri =404;
            fastcgi_pass 127.0.0.1:9000;
 
            fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
            include        fastcgi_params;
}
```

我们会发现一个叫 fastcgi_param的可配置参数，而nginx一般提供了默认值。
打开nginx目录下的fastcgi_params文件可以看到：

```
fastcgi_param  QUERY_STRING       $query_string;
fastcgi_param  REQUEST_METHOD     $request_method;
fastcgi_param  CONTENT_TYPE       $content_type;
fastcgi_param  CONTENT_LENGTH     $content_length;
fastcgi_param  SCRIPT_NAME        $fastcgi_script_name;
fastcgi_param  REQUEST_URI        $request_uri;
fastcgi_param  DOCUMENT_URI       $document_uri;
fastcgi_param  DOCUMENT_ROOT      $document_root;
fastcgi_param  SERVER_PROTOCOL    $server_protocol;
fastcgi_param  REQUEST_SCHEME     $scheme;
fastcgi_param  HTTPS              $https if_not_empty;
fastcgi_param  GATEWAY_INTERFACE  CGI/1.1;
fastcgi_param  SERVER_SOFTWARE    nginx/$nginx_version;
fastcgi_param  REMOTE_ADDR        $remote_addr;
fastcgi_param  REMOTE_PORT        $remote_port;
fastcgi_param  SERVER_ADDR        $server_addr;
fastcgi_param  SERVER_PORT        $server_port;
fastcgi_param  SERVER_NAME        $server_name;
```

这时候，我们对比下$_SERVER中http_XXX_XXX的，这不是都是http自带的头部嘛？

综合下来，这就是一个http请求完整转发到php的过程，一次完整的cgi请求，而fastcgi呢，就是对cgi协议的常驻封装，主要解决的问题是：

优化性能（cgi不停fork新进程，每个进程都干很多同样的事【加载配置文件、加载扩展】）
进程池维护,异步模型，请求是分发到空闲子进程，方便创建/销毁进程。


# fcgi协议组成（go语言描述）

### fcgi共12种消息类型
```
const (
 typeBeginRequest uint8 = 1
 typeAbortRequest uint8 = 2
 typeEndRequest uint8 = 3
 typeParams uint8 = 4
 typeStdin uint8 = 5
 typeStdout uint8 = 6
 typeStderr uint8 = 7
 typeData uint8 = 8
 typeGetValues uint8 = 9
 typeGetValuesResult uint8 = 10
 typeUnknownType uint8 = 11
)
```
见名知意：

1~3是信号类型

4~10是不同的数据类型

11、default 是错误类型

### 数据传输

通常协议传输，为了考虑数据大小的原因，都会将数据进行分片处理，fcgi也同样。

fcgi协议的每个包都由包头和包体组成

```
//It's fcgi header
type header struct {
	Version       uint8
 	Type          uint8
 	Id            uint16
 	ContentLength uint16
 	PaddingLength uint8
 	Reserved      uint8
}
```
包的正文，就是由分割的二进制内容组成，一个包最多传输2^16次方减一，也就是65535个字节。



一次完整的请求过程

请求：解析http请求获取配置->发送cgi请求->typeBeginRequest -> write Params -> write Stdin(if exists) -> write Data (if exists)
```
//写头部
if request.KeepConn {
   //if it's keep-alive
   //set flags 1
   err = cgi.writeBeginRequest(reqId, roleResponder, 1)
} else {
   err = cgi.writeBeginRequest(reqId, roleResponder, 0)
}
 
if err != nil {
   return
}
 
//写http header
err = cgi.writePairs(typeParams, reqId, request.Params)
if err != nil {
   return
}
 
 
//读取http body并写到cgi
p := make([]byte, 1024)
n, _ := request.Stdin.Read(p)
err = cgi.writeRecord(typeStdin, reqId, p[:n])
//写其他信息
 
//写最后一个换行
err = cgi.writeRecord(typeStdin, reqId, nil)
```

接收：check response type-> 错误处理(stderr、unknownType、default)->正确类型->接受数据复用socket返回给请求方。
rec := &record{} //建立一个记录缓冲
```
    var err1 error
 
    // recive untill EOF or FCGI_END_REQUEST
    for {
        err1 = rec.read(cgi.rwc)   //从cgi建立的链接读取数据
        if err1 != nil {           //判断是否有错误，判断错误是否为终止符
            if err1 != io.EOF {  // keepalive on的时候 不会有终止符  这个要注意，读完长度为content-length即当前请求返回
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



