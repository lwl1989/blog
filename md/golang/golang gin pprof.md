CreateTime:2020-05-13 18:10:23.0

# GO性能分析

Go语言内置了获取程序运行数据的工具，包括以下两个标准库：
- runtime/pprof: 采集工具型应用运行数据进行分析
- net/http/pprof: 采集服务型应用运行时数据进行分析

## pprof是什么？

pprof 是用于可视化和分析性能分析数据的工具

pprof 以 profile.proto 读取分析样本的集合，并生成报告以可视化并帮助分析数据（支持文本和图形报告）

profile.proto 是一个 Protocol Buffer v3 的描述文件，它描述了一组 callstack 和 symbolization 信息， 作用是表示统计分析的一组采样的调用栈，是很常见的 stacktrace 配置文件格式

支持什么使用模式
- Report generation：报告生成
- Interactive terminal use：交互式终端使用
- Web interface：Web 界面

可以做什么？

- CPU Profiling：CPU 分析，按照一定的频率采集所监听的应用程序 CPU（含寄存器）的使用情况，可确定应用程序在主动消耗 CPU 周期时花费时间的位置
- Memory Profiling：内存分析，在应用程序进行堆分配时记录堆栈跟踪，用于监视当前和历史内存使用情况，以及检查内存泄漏
- Block Profiling：阻塞分析，记录 goroutine 阻塞等待同步（包括定时器通道）的位置
- Mutex Profiling：互斥锁分析，报告互斥锁的竞争情况

# Gin如何做性能测试

## gin是什么

gin,ojbk就是一套http框架（应该说只是组件吧）

## gin pprof

我们上面看了pprof只有runtime和http下面有，那gin咋办？

不着急，找找就发现了，果然：

	go get https://github.com/gin-contrib/pprof

在github找到了相关demo:

![](https://oscimg.oschina.net/oscnet/up-7f109d383684dbf7cfe402d005db5495a2a.png)

相当于是注册了gin。

运行之后，有如下：

![](https://oscimg.oschina.net/oscnet/up-428e3daf59546093507454c228da162ee72.png)

打开：http://localhost:8080/debug/pprof/  (我这里是8083)

![](https://oscimg.oschina.net/oscnet/up-100fb1f557c7469e4b3d7e197e17c985199.png)

这时候你就可以进行压测和数据采集了。go pprof支持命令行。

	go tool pprof --seconds 20 http://localhost:3000/debug/pprof/goroutine
	go tool pprof http://localhost:3000/debug/pprof/goroutine?second=20

## go tool pprof命令行交互界面

go tool pprof通过命令行也可以实现强大的功能，我们来看下命令吧！

	top 默认查看程序中占用cpu前10位的函数。
	list + 函数名命令查看具体的函数分析
	pdf 可以生成可视化的pdf文件。

# 那么开始压测吧

1. 开启我们的服务

	go run main.go

2. 压测工具

推荐使用https://github.com/wg/wrk 或 https://github.com/adjust/go-wrk

我使用的是go-wrk

```
git clone git://github.com/adeven/go-wrk.git
cd go-wrk
go build
./go-wrk  -c=100 -t=8 -n=1000 http://localhost:8083/info/325720270784552960/1
```

3. 观察

![](https://oscimg.oschina.net/oscnet/up-81524872da4ecc458c18980b6d2962d04ab.png)

通过观察我们发现主要性能消耗是在runtime、net、sql

到此进行了一次完整的pprof gin观测

# 注意

- 我们只应该在性能测试的时候才在代码中引入pprof