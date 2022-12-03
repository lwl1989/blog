CreateTime:2019-11-28 16:23:21.0

# 代理

接到一个需求，要做动态ip代理，嗯，就上吧。

## go http proxy

go的http包已经包含了代理功能,使用golang net/http库发送http请求，最后都是调用 transport的 RoundTrip方法中。

```
type RoundTripper interface {
	RoundTrip(*Request) (*Response, error)
}
```

RoundTrip代表一个http事务，给一个请求返回一个响应。RoundTripper必须是并发安全的。RoundTripper接口的实现Transport结构体在源码包net/http/transport.go 中。

```
type Transport struct {
	idleMu       sync.Mutex
	closeIdle    bool                                // user has requested to close all idle conns
	idleConn     map[connectMethodKey][]*persistConn // most recently used at end
	idleConnWait map[connectMethodKey]wantConnQueue  // waiting getConns
	idleLRU      connLRU
	reqMu       sync.Mutex
	reqCanceler map[*Request]func(error)
	altMu    sync.Mutex   // guards changing altProto only
	altProto atomic.Value // of nil or map[string]RoundTripper, key is URI scheme
	connsPerHostMu   sync.Mutex
	connsPerHost     map[connectMethodKey]int
	connsPerHostWait map[connectMethodKey]wantConnQueue // waiting getConns
	Proxy func(*Request) (*url.URL, error)
	DialContext func(ctx context.Context, network, addr string) (net.Conn, error)
	Dial func(network, addr string) (net.Conn, error)
	DialTLS func(network, addr string) (net.Conn, error)
	TLSClientConfig *tls.Config
	TLSHandshakeTimeout time.Duration
	DisableKeepAlives bool
	DisableCompression bool
	MaxIdleConns int
	MaxIdleConnsPerHost int
	MaxConnsPerHost int
	IdleConnTimeout time.Duration
	ResponseHeaderTimeout time.Duration
	ExpectContinueTimeout time.Duration
	TLSNextProto map[string]func(authority string, c *tls.Conn) RoundTripper
	ProxyConnectHeader Header
	MaxResponseHeaderBytes int64
	WriteBufferSize int
	ReadBufferSize int
	nextProtoOnce      sync.Once
	h2transport        h2Transport 
	tlsNextProtoWasNil bool
	ForceAttemptHTTP2 bool
}
```


结构体字段有些多啊，看起来很累是不是？

ok，别的暂且不管，咱要的是动态ip代理的形式，因此：

	http2的忽略
	keepalive的忽略
	连接数控制的忽略

然后，因为目标的大小未知，所以

	buffersize忽略

那么剩下，我们还用看啥？

```
Proxy func(*Request) (*url.URL, error)
```
实现他就完事了

```
// ProxyURL returns a proxy function (for use in a Transport)
// that always returns the same URL.
func ProxyURL(fixedURL *url.URL) func(*Request) (*url.URL, error) {
	return func(*Request) (*url.URL, error) {
		return fixedURL, nil
	}
}
```
然后我找到这个方法

```
proxyUrl, _ := url.Parse("scheme://example.com")
client := http.Client{Transport: &http.Transport{Proxy: http.ProxyURL(proxyUrl)}}
```

这样请求就会简单的转发到 scheme://example.com 了

## 细节

http2注意：
	在Go1.6版本及之后，HTTP2会自动开启，当且仅当：

	请求是基于TLS／HTTPS的
	Server.TLSNextProto设置为nil(注意，如果设置为空map，那会禁用HTTP2)
	Server.TLSConfig被设置并且ListenAndServerTLS被使用；或者，使用Serve,同时tls.Config.NextProtos包含了"h2"，例如[]string{"h2","http/1.1"}
	同时在Go1.8版本修复了一个关于HTTP2的ReadTimeout的Bug，再结合1.8的其它特性，我的建议是尽快升级1.8。


# http返回乱码问题

```
 response, err := client.Do(req)
    if err != nil {
        return
    }

    defer response.Body.Close()

if content, err := ioutil.ReadAll(reader); err != nil {
     //todo: log
}

fmt.Println(string(content[:]))
```

啥，打印字符串出现乱码？百度google一日游吧。

	�9��7�� ���չ�������HTTP����IP 

预分析原因：

1. 编码不对（原来做java和Php常见）
2. 数据被截取

go默认是Utf-8的，然后我用postman测试了一下，编码是正确的，排除1.

然后数据被截取？的确是可能存在的，因此，我拿出了content-length然后len(content)，结果还是一样。

这就奇怪了。

思来想去，我突然想起，nginx正式机上都会加上gzip压缩，我试了下拿出返回头。

```
assert(response.Header.Get("Content-Encoding") == "gzip")
```

果然没错

这时候我们想拿到真正的内容，就得先对body进行处理了。

```
 reader := response.Body
    if response.Header.Get("Content-Encoding") == "gzip" {
        reader, err = gzip.NewReader(response.Body)
        if err != nil {
            //todo:log
        }
    }
  content, err = ioutil.ReadAll(reader);
```

