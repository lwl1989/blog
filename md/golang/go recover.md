CreateTime:2019-05-21 14:54:40.0

## 异常、错误常见语言处理

一般语言都有其错误处理方式，就以鄙人熟悉的php来距离吧。

PHP有多个级别的错误处理方式，以防止程序在还未正确执行完毕时，就造成了程序的提前结束。

1. try/catch/finally

2. error handler

3. exit handler

一般现代语言和框架中都有详细的应用位置：

比如
```
public function createSocket()
    {
        set_error_handler([self::class, 'nullErrorHandler']);
        try {
            return stream_socket_client($this-&gt;host, $errno, $errstr, 3, STREAM_CLIENT_CONNECT | STREAM_CLIENT_ASYNC_CONNECT);
        } finally {
            restore_error_handler();
        }
    }
```

看代码我们可以很明确获知，当我们创建一个流客户端失败的情况，会被error_hander注册的方法捕获，然后进行一系列的逻辑处理。成功则注销掉（不一定要注销，根据需求产出是最重要的）。

## go中是如何对错误进行处理的

go的错误处理，好像一直是被大家吐槽的点（据说go2.0已经在改进了，不过官方说的要保持良好的向前兼容性，我对此持怀疑态度)。

go的错误处理分成两种

1. err

2. panic

通常err都是逻辑错误产生，然后，嗯，就是被人吐槽的那点，需要层层递归返回，经过多层调用栈，甚至可能在中途被_抛弃，然后丢失在茫茫人海中。

而panic是属于致命错误，这个时候，会导致go当前线程结束运行（runtime error）。

兄弟，这好像不合理啊，如果线程异常退出就gg了，那我们的程序肯定是得不到正确的结果了。

## go中利用recover实现错误捕获

go中提供了一个recover方法

```
// The recover built-in function allows a program to manage behavior of a
// panicking goroutine. Executing a call to recover inside a deferred
// function (but not any function called by it) stops the panicking sequence
// by restoring normal execution and retrieves the error value passed to the
// call of panic. If recover is called outside the deferred function it will
// not stop a panicking sequence. In this case, or when the goroutine is not
// panicking, or if the argument supplied to panic was nil, recover returns
// nil. Thus the return value from recover reports whether the goroutine is
// panicking.
func recover() interface{}
```
简而言之，就是只要panic被recover所捕获，那么panic之后的代码还是能继续运行下去。

看一段代码

```
func DoTicker(event Event) {
    timer := time.NewTicker(2 * time.Second)
    defer func() {
        // 如果程序异常, 停止当前定时任务,记录日志,重启任务
        if x := recover(); x != nil {
            timer.Stop()
            log.Errorf(event.GetName()+" panic :", x)
            go DoTicker(event)
        }
    }()
    for {
        // 监听IO
        select {
        // 如果时间通道数据读取成功,
        case &lt;-timer.C:
            log.Debug(event.GetName())
            event.Do()
        }
    }
}
```

很明显，这里是一个定时器任务，需要2秒执行一次任务。

当任务执行失败的时候，则被recover捕获到。

一般我们顺序执行的话，那么就结束了。

但是panic是逆向回溯调用栈的，所以肯定的是，当前方法会调用defer方法。

如果我们在defer中，判定到当前方法是遭遇到panic退出的时候，开启一个新的调用栈继续开始任务，就实现了任务的自动重启。

那么，分析一下这段代码的问题所在：

1. 糟糕情况将无限panic循环重启任务
2. 未对异常情况进行处理

所以，在未明确确定panic来源的时候，尽量是不推荐用recover保证程序的继续执行。



