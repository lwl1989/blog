CreateTime:2018-05-23 00:21:15.0

# golang中定时器

golang中提供了2种定时器timer和ticker（如果JS很熟悉的话应该会很了解），分别是一次性定时器和重复任务定时器。

### 一般用法:

```
func main() {
 
    input := make(chan interface{})
 
    //producer - produce the messages
    go func() {
        for i := 0; i < 5; i++ {
            input <- i
        }
        input <- "hello, world"
    }()
 
    t1 := time.NewTimer(time.Second * 5)
    t2 := time.NewTimer(time.Second * 10)
 
    for {
        select {
        //consumer - consume the messages
        case msg := <-input:
            fmt.Println(msg)
 
        case <-t1.C:
            println("5s timer")
            t1.Reset(time.Second * 5)
 
        case <-t2.C:
            println("10s timer")
            t2.Reset(time.Second * 10)
        }
    }
}
```

### 源码观察

这个C是啥，我们去源码看看，以timer为例：

```
type Timer struct {
	C <-chan Time
	r runtimeTimer
}
```
原来是一个channel，其实有GO基础的都知道，GO的运算符当出现的->或者<-的时候，必然是有一端是指channel。按照上面的例子来看，就是阻塞在一个for循环内，等待到了定时器的C从channel出来，当获取到值的时候，进行想要的操作。

# 设计我们的定时任务队列

### 我的需求

当时我的需求是这样，我需要接收到客户端的请求并产生一个定时任务，会在固定时间执行，可能是一次，也可能是多次，也可能到指定时间自动停止，可能当任务终止的时候，我还要能停止掉。

具体我画了个流程图，差不多如下，画图水平有限，请见谅。

![流程图](https://static.oschina.net/uploads/img/201805/23002031_uvVk.png "流程图")


### 定义结构

```
type OnceCron struct {
	tasks  []*Task          //任务的列队
	add    chan *Task       //当遭遇到新任务的时候
	remove chan string       //当遭遇到删除任务的时候
	stop   chan struct{}      //当遇到停止信号的时候
	Logger *log.Logger      //日志  
}
type Job interface {
	Run()                  //执行接口
}
type Task struct {
        Job     Job            //要执行的任务 
	Uuid    string           //任务标识,删除时用
	RunTime int64           //执行时间
	Spacing int64           //间隔时间
	EndTime int64           //结束时间
	Number  int             //总共要次数
}
```

### 队列实现

首先，我们要获得一个队列任务

func NewCron() *OnceCron 常规操作，为了节省篇幅，我就不写出来，具体可以看源码，贴在了底部。

然后，开始定时器队列的运行，一般，都会命名为Start。那么就有一个问题，我们刚开始启动程序的时候，这个时候是没有任务队列，那岂不是for{ select{}}在等待个毛毛球？所以，我们需要在Start的时候添加一个默认的任务，
我是这么做的，添加了一个一小时执行一次的重复队列，防止队列退出。

```
func (one *OnceCron) Start() {
	//初始化的時候加入一個一年的長定時器,間隔1小時執行一次
	task := getTaskWithFuncSpacing(3600,  time.Now().Add(time.Hour*24*365).Unix（） , func() {
		log.Println("It's a Hour timer!")
	}) //为了代码格式markdown 里面有个括号我改成全角了
	one.tasks = append(one.tasks, task)
	go one.run()  //协成执行 防止主进程被阻塞
}
```

执行部分应该是重点的，我的理解是，分成三部:

1. 首先获得一个最先执行的任务
2. 然后产生一个定时器，用于执行任务
3. 进行阻塞判断，获取我们要进行的操作

```
func (one *OnceCron) run() {

	for {
                //第一步 获取任务
		now := time.Now()   //获取到当前时间
		task, key := one.GetTask() //获取最近的一个任务的执行时间
		i64 := task.RunTime - now.Unix() //任务执行和当前时间的差

		var d time.Duration
		if i64 < 0 {   //如果任务时间已过期,将执行时间改成现在并且利马执行
			one.tasks[key].RunTime = now.Unix()  
			one.doAndReset(key)
                        continue
		} else {  //否则，获取距离执行开始的间隔时间
			d = time.Unix(task.RunTime, 0).Sub(now)
		}
                //第二步 产生定时器
		timer := time.NewTimer(d)  

		//第三步 捕获定时器或者其他事件
		for {
			select {	
                        //当定时器到了执行时间时，执行当前任务并关闭定时器
			case <-timer.C:
				one.doAndReset(key)
				if task != nil {
					go task.Job.Run()
					timer.Stop()
				}

			//当外部添加了任务时，关闭当前定时器
			case <-one.add:
				timer.Stop()
			//当外部要删除一个任务时，删除ID为uuidstr的任务
			case uuidstr := <-one.remove:
				one.removeTask(uuidstr)
				timer.Stop()
			//当遇到要关闭整个定时器任务时
			case <-one.stop:
				timer.Stop()
				return
			}

			break
		}
	}
}
```

# 后记

这个文章纯粹为笔记分析类的文章，旨在分析我碰到一个需求是如何通过分析过程来产生我们需要的代码的。

源码地址:

[timing 一个任务队列](https://github.com/lwl1989/timing)

应用地址:

[一个应用于谷歌消息推送的转发中间件](https://github.com/lwl1989/spinx)

参考源码:

[GOLANG实现crontab功能](https://github.com/robfig/cron)