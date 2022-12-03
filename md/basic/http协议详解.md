CreateTime:2016-06-23 15:10:32.0

### URL
http（超文本传输协议）是一个基于请求与响应模式的、无状态的、应用层的协议，常基于TCP的连接方式，HTTP1.1版本中给出一种持续连接的机制，绝大多数的Web开发，都是构建在HTTP协议之上的Web应用。
HTTP URL (URL是一种特殊类型的URI，包含了用于查找某个资源的足够的信息)的格式如下：
http://host[":"port][abs_path]
http表示要通过HTTP协议来定位网络资源；host表示合法的Internet主机域名或者IP地址；port指定一个端口号，为空则使用缺省端口80；abs_path指定请求资源的URI；如果URL中没有给出abs_path，那么当它作为请求URI时，必须以“/”的形式给出，通常这个工作浏览器自动帮我们完成。

### Request
请求行由三个标记组成：请求方法、请求 URI 和 HTTP 版本，它们用空格分隔。
例如：GET /news.asp HTTP/1.1
消息头（Message Headers）

由域名/值对组成，每行一对，域名和值之间用紧跟的英文冒号（“:”），单空格（SP）分开。消息头通知服务器有关于客户端的功能和标识。如 Host: http://demo.com:80 表示请求的资源所在的主机和端口号，在 HTTP/1.1 协议中，Host 消息头是必选的。还可以有其他一些如 Accept-Charset、Accept-Encoding、Authorization 等等，详见 RFC1945，RFC2616。
消息体（Entity Body）

HTTP 消息的消息体（如果存在），用于携带与请求相关联的数据，例如可以存一些请求需要的参数等。由消息头中的 Content-Length 或 Transfer-Encoding 来指示。消息头里的 Content-Type 说明了数据的传输类型。
一个完整的带消息体的 HTTP 请求示例如下：
POST /news.asp HTTP/1.1
Host: demo.com:80
Content-Length: 15
 
[a=1,(b=2,c=3)]

### Response
HTTP响应也是由三个部分组成，分别是：状态行、消息报头、响应正文
1、状态行格式如下：
HTTP-Version Status-Code Reason-Phrase CRLF
其中，HTTP-Version表示服务器HTTP协议的版本；Status-Code表示服务器发回的响应状态代码；Reason-Phrase表示状态代码的文本描述。
状态代码有三位数字组成，第一个数字定义了响应的类别，且有五种可能取值：
1xx：指示信息--表示请求已接收，继续处理
2xx：成功--表示请求已被成功接收、理解、接受
3xx：重定向--要完成请求必须进行更进一步的操作
4xx：客户端错误--请求有语法错误或请求无法实现
5xx：服务器端错误--服务器未能实现合法的请求
常见状态代码、状态描述、说明：
200 OK      //客户端请求成功
400 Bad Request  //客户端请求有语法错误，不能被服务器所理解
401 Unauthorized //请求未经授权，这个状态代码必须和WWW-Authenticate报头域一起使用 
403 Forbidden  //服务器收到请求，但是拒绝提供服务
404 Not Found  //请求资源不存在，eg：输入了错误的URL
500 Internal Server Error //服务器发生不可预期的错误
503 Server Unavailable  //服务器当前不能处理客户端的请求，一段时间后可能恢复正常
eg：HTTP/1.1 200 OK （CRLF）
2、响应报头后述
3、响应正文就是服务器返回的资源的内容 

### 通用头
通用头域
通用头域包含请求和响应消息都支持的头域，通用头域包含Cache-Control、Connection、Date、Pragma、Transfer-Encoding、Upgrade、Via。对通用头域的扩展要求通讯双方都支持此扩展，如果存在不支持的通用头域，一般将会作为实体头域处理。下面简单介绍几个在UPnP消息中使用的通用头域：
1.Cache-Control头域
Cache-Control指定请求和响应遵循的缓存机制。在请求消息或响应消息中设置Cache-Control并不会修改另一个消息处理过程中的缓存处理过程。请求时的缓存指令包括no-cache、no-store、max-age、max-stale、min-fresh、only-if-cached，响应消息中的指令包括public、private、no-cache、no-store、no-transform、must-revalidate、proxy-revalidate、max-age。各个消息中的指令含义如下：
Public指示响应可被任何缓存区缓存。
Private指示对于单个用户的整个或部分响应消息，不能被共享缓存处理。这允许服务器仅仅描述当用户
http结构
http结构
的部分响应消息，此响应消息对于其他用户的请求无效。
no-cache指示请求或响应消息不能缓存
no-store用于防止重要的信息被无意的发布。在请求消息中发送将使得请求和响应消息都不使用缓存。
max-age指示客户机可以接收生存期不大于指定时间（以秒为单位）的响应。
min-fresh指示客户机可以接收响应时间小于当前时间加上指定时间的响应。
max-stale指示客户机可以接收超出超时期间的响应消息。如果指定max-stale消息的值，那么客户机可以接收超出超时期指定值之内的响应消息。
HTTP Keep-Alive
Keep-Alive功能使客户端到服务器端的连接持续有效，当出现对服务器的后继请求时，Keep-Alive功能避免了建立或者重新建立连接。市场上的大部分Web服务器，包括iPlanet、IIS和Apache，都支持HTTP Keep-Alive。对于提供静态内容的网站来说，这个功能通常很有用。但是，对于负担较重的网站来说，这里存在另外一个问题：虽然为客户保留打开的连接有一定的好处，但它同样影响了性能，因为在处理暂停期间，本来可以释放的资源仍旧被占用。当Web服务器和应用服务器在同一台机器上运行时，Keep- Alive功能对资源利用的影响尤其突出。
KeepAliveTime 值控制 TCP/IP 尝试验证空闲连接是否完好的频率。如果这段时间内没有活动，则会发送保持活动信号。如果网络工作正常，而且接收方是活动的，它就会响应。如果需要对丢失接收方敏感，换句话说，需要更快地发现丢失了接收方，请考虑减小这个值。如果长期不活动的空闲连接出现次数较多，而丢失接收方的情况出现较少，您可能会要提高该值以减少开销。缺省情况下，如果空闲连接 7200000 毫秒（2 小时）内没有活动，Windows 就发送保持活动的消息。通常，1800000 毫秒是首选值，从而一半的已关闭连接会在 30 分钟内被检测到。 KeepAliveInterval 值定义了如果未从接收方收到保持活动消息的响应，TCP/IP 重复发送保持活动信号的频率。当连续发送保持活动信号、但未收到响应的次数超出 TcpMaxDataRetransmissions 的值时，会放弃该连接。如果期望较长的响应时间，您可能需要提高该值以减少开销。如果需要减少花在验证接收方是否已丢失上的时间，请考虑减小该值或 TcpMaxDataRetransmissions 值。缺省情况下，在未收到响应而重新发送保持活动的消息之前，Windows 会等待 1000 毫秒（1 秒）。 KeepAliveTime 根据你的需要设置就行，比如10分钟，注意要转换成MS。 XXX代表这个间隔值得大小。
2.Date头域
Date头域表示消息发送的时间，时间的描述格式由rfc822定义。例如，Date:Mon,31Dec200104:25:57GMT。Date描述的时间表示世界标准时，换算成本地时间，需要知道用户所在的时区。
3.Pragma头域
Pragma头域用来包含实现特定的指令，最常用的是Pragma:no-cache。在HTTP/1.1协议中，它的含义和Cache-Control:no-cache相同。
请求消息
请求消息的第一行为下面的格式：
MethodSPRequest-URISPHTTP-VersionCRLFMethod表示对于Request-URI完成的方法，这个字段是大小写敏感的，包括OPTIONS、GET、HEAD、POST、PUT、DELETE、TRACE。方法GET和HEAD应该被所有的通用WEB服务器支持，其他所有方法的实现是可选的。GET方法取回由Request-URI标识的信息。HEAD方法也是取回由Request-URI标识的信息，只是可以在响应时，不返回消息体。POST方法可以请求服务器接收包含在请求中的实体信息，可以用于提交表单，向新闻组、BBS、邮件群组和数据库发送消息。
SP表示空格。Request-URI遵循URI格式，在此字段为星号（*）时，说明请求并不用于某个特定的资源地址，而是用于服务器本身。HTTP-Version表示支持的HTTP版本，例如为HTTP/1.1。CRLF表示换行回车符。请求头域允许客户端向服务器传递关于请求或者关于客户机的附加信
http架构
http架构
息。请求头域可能包含下列字段Accept、Accept-Charset、Accept-Encoding、Accept-Language、Authorization、From、Host、If-Modified-Since、If-Match、If-None-Match、If-Range、If-Range、If-Unmodified-Since、Max-Forwards、Proxy-Authorization、Range、Referer、User-Agent。对请求头域的扩展要求通讯双方都支持，如果存在不支持的请求头域，一般将会作为实体头域处理。
典型的请求消息：
Host: download.*******.de
Accept: */*
Pragma: no-cache
Cache-Control: no-cache
User-Agent: Mozilla/4.04[en](Win95;I;Nav)
Range: bytes=554554-
上例第一行表示HTTP客户端（可能是浏览器、下载程序）通过GET方法获得指定URL下的文件。棕色的部分表示请求头域的信息，绿色的部分表示通用头部分。
1.Host头域
Host头域指定请求资源的Intenet主机和端口号，必须表示请求url的原始服务器或网关的位置。HTTP/1.1请求必须包含主机头域，否则系统会以400状态码返回。
2.Referer头域
Referer头域允许客户端指定请求uri的源资源地址，这可以允许服务器生成回退链表，可用来登陆、优化cache等。他也允许废除的或错误的连接由于维护的目的被追踪。如果请求的uri没有自己的uri地址，Referer不能被发送。如果指定的是部分uri地址，则此地址应该是一个相对地址。
3.Range头域
Range头域可以请求实体的一个或者多个子范围。例如，
表示头500个字节：bytes=0-499
表示第二个500字节：bytes=500-999
表示最后500个字节：bytes=-500
表示500字节以后的范围：bytes=500-
第一个和最后一个字节：bytes=0-0,-1
同时指定几个范围：bytes=500-600,601-999
但是服务器可以忽略此请求头，如果无条件GET包含Range请求头，响应会以状态码206（PartialContent）返回而不是以200（OK）。
4.User-Agent头域
User-Agent头域的内容包含发出请求的用户信息。
响应消息
响应消息的第一行为下面的格式：
HTTP-VersionSPStatus-CodeSPReason-PhraseCRLF
HTTP-Version表示支持的HTTP版本，例如为HTTP/1.1。Status-Code是一个三个数字的结果代码。Reason-Phrase给Status-Code提供一个简单的文本描述。Status-Code主要用于机器自动识别，Reason-Phrase主要用于帮助用户理解。Status-Code的第一个数字定义响应的类别，后两个数字没有分类的作用。第一个数字可能取5个不同的值：
1xx:信息响应类，表示接收到请求并且继续处理
2xx:处理成功响应类，表示动作被成功接收、理解和接受
3xx:重定向响应类，为了完成指定的动作，必须接受进一步处理
4xx:客户端错误，客户请求包含语法错误或者是不能正确执行
5xx:服务端错误，服务器不能正确执行一个正确的请求
响应头域允许服务器传递不能放在状态行的附加信息，这些域主要描述服务器的信息和Request-URI进一步的信息。响应头域包含Age、Location、Proxy-Authenticate、Public、Retry-After、Server、Vary、Warning、WWW-Authenticate。对响应头域的扩展要求通讯双方都支持，如果存在不支持的响应头域，一般将会作为实体头域处理。
典型的响应消息：
HTTP/1.0200OK
Date:Mon,31Dec200104:25:57GMT
Server:Apache/1.3.14(Unix)
Content-type:text/html
Last-modified:Tue,17Apr200106:46:28GMT
Etag:"a030f020ac7c01:1e9f"
Content-length:39725426
Content-range:bytes55******/40279980
上例第一行表示HTTP服务端响应一个GET方法。棕色的部分表示响应头域的信息，绿色的部分表示通用头部分，红色的部分表示实体头域的信息。
1.Location响应头
Location响应头用于重定向接收者到一个新URI地址。
2.Server响应头
Server响应头包含处理请求的原始服务器的软件信息。此域能包含多个产品标识和注释，产品标识一般按照重要性排序。