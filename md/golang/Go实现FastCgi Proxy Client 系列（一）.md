CreateTime:2018-04-03 00:33:16.0

# 什么是FastCgi

再了解FastCgi之前，我们一定要先知道，什么叫Cgi。

> CGI全称是“通用网关接口”(Common Gateway Interface)，HTTP服务器与你的或其它机器上的程序进行“交谈”的一种工具，其程序一般运行在网络服务器上。 CGI可以用任何一种语言编写，只要这种语言具有标准输入、输出和环境变量。如php,perl,tcl等。

### cgi的弊端

cgi会产生什么问题呢？

当每一个请求进入的时候，cgi都会fork一个新的进程，然后以php为例，每个请求都要耗费相当大的内存，这样一来，并发起来，完全就会GG。

为了解决这个问题，于是产生了fastCgi。

### 对比

盗2张百度的图

![cgi](https://static.oschina.net/uploads/img/201804/02232200_nbPg.jpg "在这里输入图片标题")

![fastCgi](https://static.oschina.net/uploads/img/201804/02232215_XRtu.jpg "在这里输入图片标题")

从图中可以看出，FastCgi解决的就是这一痛点，当一个新的请求进来的时候，会交给一个已经产生的进程进行处理，而不是fork新的东西出来（php 7还有更多优化，欢迎大家进坑，比如opcached）。

# FastCgi协议组成

我目前了解的协议都是由三个部分组成：协议头、正文、结束符（或者说协议尾部），现在我们就来分析下这个协议。

主要参考资料来源于:[麻省理工文档](http://www.mit.edu/~yandros/doc/specs/fcgi-spec.html)

### FastCgi协议头

协议有这几种消息类型
```
#define FCGI_BEGIN_REQUEST 1
#define FCGI_ABORT_REQUEST 2
#define FCGI_END_REQUEST 3
#define FCGI_PARAMS 4
#define FCGI_STDIN 5
#define FCGI_STDOUT 6
#define FCGI_STDERR 7
#define FCGI_DATA 8
#define FCGI_GET_VALUES 9
#define FCGI_GET_VALUES_RESULT 10
#define FCGI_UNKNOWN_TYPE 11
#define FCGI_MAXTYPE (FCGI_UNKNOWN_TYPE)
```
很明显，如果我们要发起一个请求，必然要使用消息类型FCGI_BEGIN_REQUEST的进行请求。

看一段通用的包头。
```
typedef struct {
    unsigned char roleB1;
    unsigned char roleB0;
    unsigned char flags;
    unsigned char reserved[5];
} FCGI_BeginRequestBody;

typedef struct {
    FCGI_Header header;
    FCGI_BeginRequestBody body;
} FCGI_BeginRequestRecord;

typedef struct {
    unsigned char appStatusB3;
    unsigned char appStatusB2;
    unsigned char appStatusB1;
    unsigned char appStatusB0;
    unsigned char protocolStatus;
    unsigned char reserved[3];
} FCGI_EndRequestBody;
```

我们可以把这个过程看成 begin->(header and content)->end。header这个很有意思，不过联系一下http协议的header,自然一下就了解了，其实在cgi协议里，header应该也属于正文的一种，只是类型比较特殊。

# golang的实现

假如我们要实现一个golang fastcgi proxy client的部分，那自然，首先我们要先讲http请求解析成cgi协议的形式。

### beginRequest
    
分析协议得出，web服务器向FastCGI程序发送一个 8 字节 type=FCGI_BEGIN_REQUEST的消息头和一个8字节  FCGI_BeginRequestBody 结构的 消息体，标志一个新请求的开始。

再仔细一想，这不对，服务端是如何知道我们发送的是啥，接受到的数据该如何解析。

 **从下面开始，我都会用golang来显示代码。**

所以，这时候我们需要一个包头。fastCgi通用的包头(此处的header不是http header)为：
```
type header struct {
	Version       uint8        //协议版本  默认 01
	Type          uint8        // 请求类型
	Id            uint16       // 请求id
	ContentLength uint16  //正文长度
	PaddingLength uint8  //是否有字符补齐
	Reserved      uint8    //预留
}
```
先产生我们要产生的包体
```
b := [8]byte{byte(role >> 8), byte(role), flags}
role一般默认用1   1响应器 2验证器 3过滤器
flags 标志是否保持连接 就是http的keeponlive
```
这个时候，我们就先写包头,然后写包体即可
```
func (cgi *FCGIClient) writeRecord(recType uint8, reqId uint16, content []byte) (err error) {
	cgi.mutex.Lock()
	defer cgi.mutex.Unlock()
	cgi.buf.Reset()
	cgi.h.init(recType, reqId, len(content))
        //写包头
	if err := binary.Write(&cgi.buf, binary.BigEndian, cgi.h); err != nil {
		return err
	} 
        //写包体
	if _, err := cgi.buf.Write(content); err != nil {
		return err
	}
        //写补位
	if _, err := cgi.buf.Write(pad[:cgi.h.PaddingLength]); err != nil {
		return err
	}
        //将缓冲区写到之前打开的链接  
	_, err = cgi.rwc.Write(cgi.buf.Bytes())
	return err
}
```
这里应该会很好奇，cgi.rwc是什么。这里我们预留到下一次讲，知道这里是一个建立的socket连接即可。

### 请求正文
大家都知道 http协议分成 header和body

#### http request header
 
cgi协议里有单独对header的处理，即消息类型为FCGI_PARAMS(4)。

我们要从请求里获取到这次的header，很明显,header是一个map[string]string的结构。

```
func buildEnv(documentRoot string,r *http.Request) (err error,env map[string]string){
	env = make(map[string]string)
	index := "index.php"
	filename := documentRoot+"/"+index
	if r.URL.Path == "/.env" {
		return errors.New("not allow"),env
	} else if r.URL.Path == "/" || r.URL.Path == "" {
		filename = documentRoot + "/" + index
	} else {
		filename = documentRoot + r.URL.Path
	}

	for name,value := range serverEnvironment {
		env[name] = value
	}

	//......其他mapping


	for header, values := range r.Header {
		env["HTTP_" + strings.Replace(strings.ToUpper(header), "-", "_", -1)] = values[0]
	}
	return errors.New("not allow"),env
}
```
同样，我们先发送包头，再发送包体即可。

#### http request body

包体就是我们提交的数据,比如Post,delete,put等等操作中，正文包含的数据。

同样，我们先发送包头，再发送包体即可。

 **但是需要注意的是，一般包体都会很大，但是明显，我们ContentLength只有16位的长度，很大可能是无法一次发送完毕。**

因此，我们需要分包进行发送（最大65535）。

### 请求结束 endRequest

依照上面的，同样，我们发送结束的包头和包体，修改type即可。

### 获取返回数据
    
开始我又说一个cgi.rwc是一个socket连接，数据都写往了那里，自然也要从那里读回来。

```
// recive untill EOF or FCGI_END_REQUEST
	for {
		err1 = rec.read(cgi.rwc)
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
			fallthrough
		default:
			break
		}
	}
```

一个简单的demo就完成了。这个时候，我们只需要将接收的数据格式化输出给用户即可。

# 我的github

给力点，大哥们，来点star吧。

在下一篇中，我们将讲建立连接的协议实现的细节部分。

[spinx 一个golang实现的fastcgi proxy client](https://github.com/lwl1989/spinx)