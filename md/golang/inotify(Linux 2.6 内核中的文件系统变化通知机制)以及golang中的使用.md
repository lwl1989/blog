CreateTime:2019-05-31 21:28:11.0

# inotify的前世今生

## inotify进化史

一看单词notify，我们就知道这肯定是用于通知的。

至于为啥叫inotify，这个可能我们就要问作者了(就像linux为什么叫linux一样)。inotify的作用是作用于linux的系统文件通知，事实上，在 inotify 之前已经存在一种类似的机制叫 dnotify，但是它存在许多缺陷：

1．对于想监视的每一个目录，用户都需要打开一个文件描述符，因此如果需要监视的目录较多，将导致打开许多文件描述符，特别是，如果被监视目录在移动介质上（如光盘和 USB 盘），将导致无法 umount 这些文件系统，因为使用 dnotify 的应用打开的文件描述符在使用该文件系统。

2．dnotify 是基于目录的，它只能得到目录变化事件，当然在目录内的文件的变化会影响到其所在目录从而引发目录变化事件，但是要想通过目录事件来得知哪个文件变化，需要缓存许多 stat 结构的数据。

3．Dnotify 的接口非常不友好，它使用 signal。

Inotify 是为替代 dnotify 而设计的，它克服了 dnotify 的缺陷，提供了更好用的，简洁而强大的文件变化通知机制：

1．Inotify 不需要对被监视的目标打开文件描述符，而且如果被监视目标在可移动介质上，那么在 umount 该介质上的文件系统后，被监视目标对应的 watch 将被自动删除，并且会产生一个 umount 事件。

2．Inotify 既可以监视文件，也可以监视目录。

3．Inotify 使用系统调用而非 SIGIO 来通知文件系统事件。

4．Inotify 使用文件描述符作为接口，因而可以使用通常的文件 I/O 操作select 和 poll 来监视文件系统的变化。

## inotify事件

Inotify 可以监视的文件系统事件包括：

```
IN_ACCESS，即文件被访问
IN_MODIFY，文件被 write
IN_ATTRIB，文件属性被修改，如 chmod、chown、touch 等
IN_CLOSE_WRITE，可写文件被 close
IN_CLOSE_NOWRITE，不可写文件被 close
IN_OPEN，文件被 open
IN_MOVED_FROM，文件被移走,如 mv
IN_MOVED_TO，文件被移来，如 mv、cp
IN_CREATE，创建新文件
IN_DELETE，文件被删除，如 rm
IN_DELETE_SELF，自删除，即一个可执行文件在执行时删除自己
IN_MOVE_SELF，自移动，即一个可执行文件在执行时移动自己
IN_UNMOUNT，宿主文件系统被 umount
IN_CLOSE，文件被关闭，等同于(IN_CLOSE_WRITE | IN_CLOSE_NOWRITE)
IN_MOVE，文件被移动，等同于(IN_MOVED_FROM | IN_MOVED_TO)
```

# go中使用inotify

既然内核中已经提供好了inotify的api，对于语言来说，无非是注册接口，实现，并加以使用。

## fsnotify

在github中寻觅，众里寻他千百度，他人就在 github.com/fsnotify/fsnotify

	go get -u github.com/fsnotify/fsnotify

就是如此的简单和粗暴

## 源码

观察源码应该是一个程序猿的必备知识，当一切的东西，从0-1为什么，怎么做的，都呈现在你面前的时候，你才能合理的去使用(或许这种思维也就是造成了我如今还单身的原因吧，因为女人从来不讲究为什么)。

这个包的核心结构就是在这，很明显，我们可以看到这里是用协程和通道模式组成
```
// Watcher watches a set of files, delivering events to a channel.
type Watcher struct {
	Events chan Event
	Errors chan error
	done   chan struct{} // Channel for sending a "quit message" to the reader goroutine

	kq int // File descriptor (as returned by the kqueue() syscall).

	mu              sync.Mutex        // Protects access to watcher data
	watches         map[string]int    // Map of watched file descriptors (key: path).
	externalWatches map[string]bool   // Map of watches added by user of the library.
	dirFlags        map[string]uint32 // Map of watched directories to fflags used in kqueue.
	paths           map[int]pathInfo  // Map file descriptors to path names for processing kqueue events.
	fileExists      map[string]bool   // Keep track of if we know this file exists (to stop duplicate create events).
	isClosed        bool              // Set to true when Close() is first called
}
```

那notify是在哪里被注册到系统的呢？继续追踪源码

```
const registerAdd = unix.EV_ADD | unix.EV_CLEAR | unix.EV_ENABLE
	if err := register(w.kq, []int{watchfd}, registerAdd, flags); err != nil {
		unix.Close(watchfd)
		return "", err
	}
```

可以明显看到，这里注册了一个系统事件
```
func register(kq int, fds []int, flags int, fflags uint32) error {
	changes := make([]unix.Kevent_t, len(fds))

	for i, fd := range fds {
		// SetKevent converts int to the platform-specific types:
		unix.SetKevent(&changes[i], fd, unix.EVFILT_VNODE, flags)
		changes[i].Fflags = fflags
	}

	// register the events
	success, err := unix.Kevent(kq, changes, nil, nil)
	if success == -1 {
		return err
	}
	return nil
}
```
其中很明显的是，注册了一个unix的系统事件

参数

1. kq 注册的队列的句柄号
2. fds 监听的文件句柄号
3. flags unix监听事件类型
4. fflags 事件标志, 就是之前说到的（Inotify 可以监视的文件系统事件包括）

而unix.Kevent最终调用的是系统内核的方法。

# 使用用例

为什么我会接触到这个东西呢？

因为有个需求是通过ftp传送文件，而文件变化几乎是不可预计的，最终我还要读取这里面的文件内容，并进行解析(新增或者修改原文件）

demo:
```
go func() {
        for {
            select {
            case event, ok := <-.Events:
                if !ok {
                    return
                }
                fmt.Println(event.Name, uint32(event.Op))
                log.Println("event:", event)
                if event.Op&fsnotify.Write == fsnotify.Write {
                    log.Println("modified file:", event.Name)
                }
            case err, ok := <-watcher.Errors:
                if !ok {
                    return
                }
                log.Println("error:", err)
            }
        }
    }()
```

项目还在开发中，后期我又更多心德会分享记录出来。

# 引用资源

- Linux 2.6.13的内核源码，http://www.kernel.org/pub/linux/kernel/v2.6/linux-2.6.13.tar.bz2。
- 内核文档，Documentation/filesystems/inotify.txt。
- LWN: Watching filesystem events with inotify，http://lwn.net/Articles/104343/。
- Monitor Linux file system events with inotify，http://www.ibm.com/developerworks/linux/library/l-inotify.html。
- Robert M. Love's README, http://www.kernel.org/pub/linux/kernel/people/rml/inotify/README。
- The recent inotify patch for linux 2.6.13-rc2，http://www.kernel.org/pub/linux/kernel/people/rml/inotify/v2.6/0.24/inotify-0.24-rml-2.6.13-rc2-4.patch。