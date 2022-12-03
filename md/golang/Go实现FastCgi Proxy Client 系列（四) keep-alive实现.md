CreateTime:2018-06-03 18:23:07.0

# 先贴几个链接

前三篇

[Go实现FastCgi Proxy Client 系列（三）](https://my.oschina.net/lwl1989/blog/1813102)

[Go实现FastCgi Proxy Client 系列（二）](https://my.oschina.net/lwl1989/blog/1789583)

[Go实现FastCgi Proxy Client 系列（一）](https://my.oschina.net/lwl1989/blog/1788957)

灵感帖

[TCP keepalive 和 http keep-alive](https://segmentfault.com/a/1190000012894416)

[FastCGI Specification](http://www.mit.edu/~yandros/doc/specs/fcgi-spec.html)

[Golang 优化之路——HTTP长连接](http://blog.cyeam.com/golang/2017/05/31/go-http-keepalive)

[go HTTP Client大量长连接保持](https://blog.csdn.net/kdpujie/article/details/73177179)

很感谢上面几个文章，让我突然想通了我之前一直没搞懂的问题，虽然问题不是一种，但是也算是旁敲侧击的让我茅塞顿开了，非常感谢！

# 分析

### 协议
麻省理工学院的文档上是这么描述的:

> The flags component contains a bit that controls connection shutdown:

> flags & FCGI_KEEP_CONN: If zero, the application closes the connection after responding to this request. If not zero, the application does not close the connection after responding to this request; the Web server retains responsibility for the connection.

好吧，高大上的描述。各种文档里对fastcgi协议里没怎么讲keep-alive的作用，就只是很简单的说了，如果是开启了，就能让http连接进行复用。然而按照之前的知识（我没跟上时代啊）,http协议是无状态的协议，死活我也是想不通如何连接复用。



然而我忘记了，http也是基于tcp协议的，tcp一般而言，如果你不手动进行断开，或者网络原因引起的话，是会保持一个长连接的（通常都会设置超时）。

盗张图：

![输入图片说明](https://static.oschina.net/uploads/img/201806/03180549_rb01.jpg "在这里输入图片标题")

可以看出，假如，我们不进行最后的4次挥手操作，在超时范围内，这个tcp连接是不会断开的。

### 代码

然后我再去看go官方对fastcgi协议的实现，我发现一个很重要的地方（源码位置 net/http/fastcgi/child.go）：

当用户发起终止信号的时候，keep-alive起了作用，那就是说，我的proxy层只要不进行断开连接，这个tcp连接就依然还是可用的。
```
case typeAbortRequest:
		//......
		if !req.keepConn {
			// connection will close upon return
			return errCloseConn
		}
		return nil
```

然后，我继续阅读源码，此时，我要唱一下，“终于等到你，还好我没忘记”。果然是这样，如果链接不是keep-alive，会即时关闭tcp连接。
```
func (c *child) serveRequest(req *request, body io.ReadCloser) {
	//......

	if !req.keepConn {
		c.conn.Close()
	}
}
```

# 修改实现

在我们读取数据的位置，我们进行相关操作即可，如果，请求不是keep-alive 自然，返回值会包含终止符 EOF，这时可以直接返回；反之，如果是keep-alive，则会每个请求都获得到一个typeEndRequest（值为3，具体请看[第一篇](https://my.oschina.net/lwl1989/blog/1788957)）的标识
```
// recive untill EOF or FCGI_END_REQUEST
	for {
		err1 = rec.read(cgi.rwc)
		//if !keep-alive the end has EOF
		if err1 != nil {
			if err1 != io.EOF {
				err = err1
			}
			break
		}

		switch {
		case rec.h.Type == typeStdout:
			retout = append(retout, rec.content()...)
		case rec.h.Type == typeStderr:
			reterr = append(reterr, rec.content()...)
		case rec.h.Type == typeEndRequest:
			//if keep-alive
			//It's had return
			//But connection Not close
			retout = append(retout, rec.content()...)
			return
		default:
			//fallthrough
		}
	}
```

# 总结

## 个人总结

那，其实很简单不是嘛？我之前只是理解错了typeEndRequest这个类型，我把他当成了错误的时候才会返回，报了一个fallthrough。更简单的一句就是，**复用的不是http，不是http，不是！复用的tcp!tcp!tcp!**

然后，当遇到一个自己暂时无法理解的问题时候，可能你已经钻入了死胡同。这个时候，你可以放松自己精神，玩玩游戏，运动运动都行(哈哈，这个问题我都忘记一个月了，感谢群友的问题，虽然我没帮他解决，但是他提示了我，我还要没解决的问题)。

## 下一步

下一步，将会对超时进行设置。


## spinx（小玩具）

一个实现对fastcgi协议的转发小玩具。

**Quick Start**

    go get github.com/lwl1989/spinx
    
    cd $gopath/src/github.com/lwl1989/spinx
    
    go build -o spinx main.go
    
**Install** 

    sudo ./spinx  -c=config_path install
**Remove**    

    sudo ./spinx  remove

**Start**

    sudo ./spinx start
    or
    ./spinx -d=false -c=config_path

**Stop**

    sudo ./spinx stop