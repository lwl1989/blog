CreateTime:2022-02-10 17:05:31.0

# namespace 的概念
namespace 是 Linux 内核用来隔离内核资源的方式。通过 namespace 可以让一些进程只能看到与自己相关的一部分资源，而另外一些进程也只能看到与它们自己相关的资源，这两拨进程根本就感觉不到对方的存在。具体的实现方式是把一个或多个进程的相关资源指定在同一个 namespace 中。

Linux namespaces 是对全局系统资源的一种封装隔离，使得处于不同 namespace 的进程拥有独立的全局系统资源，改变一个 namespace 中的系统资源只会影响当前 namespace 里的进程，对其他 namespace 中的进程没有影响。

# namespace 的用途

可能绝大多数的使用者和我一样，是在使用 docker 后才开始了解 linux 的 namespace 技术的。实际上，Linux 内核实现 namespace 的一个主要目的就是实现轻量级虚拟化(容器)服务。在同一个 namespace 下的进程可以感知彼此的变化，而对外界的进程一无所知。这样就可以让容器中的进程产生错觉，认为自己置身于一个独立的系统中，从而达到隔离的目的。也就是说 linux 内核提供的 namespace 技术为 docker 等容器技术的出现和发展提供了基础条件。

我们可以从 docker 实现者的角度考虑该如何实现一个资源隔离的容器。比如是不是可以通过 chroot 命令切换根目录的挂载点，从而隔离文件系统。为了在分布式的环境下进行通信和定位，容器必须要有独立的 IP、端口和路由等，这就需要对网络进行隔离。同时容器还需要一个独立的主机名以便在网络中标识自己。接下来还需要进程间的通信、用户权限等的隔离。最后，运行在容器中的应用需要有进程号(PID)，自然也需要与宿主机中的 PID 进行隔离。也就是说这六种隔离能力是实现一个容器的基础，让我们看看 linux 内核的 namespace 特性为我们提供了什么样的隔离能力：
- IPC  信号、消息、内存、POSIX标准
- NETWORK 网络相关内容
- MOUNT 文件系统
- USER 用户系统
- UTS 主机名相关
- CGROUP cgroup根目录

上表中的前六种 namespace 正是实现容器必须的隔离技术，至于新近提供的 Cgroup namespace 目前还没有被 docker 采用。相信在不久的将来各种容器也会添加对 Cgroup namespace 的支持。

# namespace 的发展历史
Linux 在很早的版本中就实现了部分的 namespace，比如内核 2.4 就实现了 mount namespace。大多数的 namespace 支持是在内核 2.6 中完成的，比如 IPC、Network、PID、和 UTS。还有个别的 namespace 比较特殊，比如 User，从内核 2.6 就开始实现了，但在内核 3.8 中才宣布完成。同时，随着 Linux 自身的发展以及容器技术持续发展带来的需求，也会有新的 namespace 被支持，比如在内核 4.6 中就添加了 Cgroup namespace。

Linux 提供了多个 API 用来操作 namespace，它们是 clone()、setns() 和 unshare() 函数，为了确定隔离的到底是哪项 namespace，在使用这些 API 时，通常需要指定一些调用参数：CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER、CLONE_NEWUTS 和 CLONE_NEWCGROUP。如果要同时隔离多个 namespace，可以使用 | (按位或)组合这些参数。同时我们还可以通过 /proc 下面的一些文件来操作 namespace。

# 查看进程所属的 namespace

从版本号为 3.8 的内核开始，/proc/[pid]/ns 目录下会包含进程所属的 namespace 信息，使用下面的命令可以查看当前进程所属的 namespace 信息：

	$ ll /proc/$$/ns

![](https://oscimg.oschina.net/oscnet/up-7f9b45d149499c97369593f7d55cef9777e.png)

首先，这些 namespace 文件都是链接文件。链接文件的内容的格式为 xxx:[inode number]。其中的 xxx 为 namespace 的类型，inode number 则用来标识一个 namespace，我们也可以把它理解为 namespace 的 ID。如果两个进程的某个 namespace 文件指向同一个链接文件，说明其相关资源在同一个 namespace 中。
其次，在 /proc/[pid]/ns 里放置这些链接文件的另外一个作用是，一旦这些链接文件被打开，只要打开的文件描述符(fd)存在，那么就算该namespace 下的所有进程都已结束，这个 namespace 也会一直存在，后续的进程还可以再加入进来。

## 挂载
除了打开文件的方式，我们还可以通过文件挂载的方式阻止 namespace 被删除。比如我们可以把当前进程中的 uts 挂载到 ~/uts 文件：

$ touch ~/uts
$ sudo mount --bind /proc/$$/ns/uts ~/uts

使用 stat 命令检查下结果：

![](https://oscimg.oschina.net/oscnet/up-82a4fa0eb33e08c45a5952c1a21534ee209.png)

很神奇吧，~/uts 的 inode 和链接文件中的 inode number 是一样的，它们是同一个文件。

## 基础函数

### clone() 函数

我们可以通过 clone() 在创建新进程的同时创建 namespace。clone() 在 C 语言库中的声明如下：

	/* Prototype for the glibc wrapper function */
	#define _GNU_SOURCE
	#include <sched.h>
	int clone(int (*fn)(void *), void *child_stack, int flags, void *arg);

实际上，clone() 是在 C 语言库中定义的一个封装(wrapper)函数，它负责建立新进程的堆栈并且调用对编程者隐藏的 clone() 系统调用。

Clone() 其实是 linux 系统调用 fork() 的一种更通用的实现方式，它可以通过 flags 来控制使用多少功能。一共有 20 多种 CLONE_ 开头的 falg(标志位) 参数用来控制 clone 进程的方方面面(比如是否与父进程共享虚拟内存等)，下面我们只介绍与 namespace 相关的 4 个参数：

- fn：指定一个由新进程执行的函数。当这个函数返回时，子进程终止。该函数返回一个整数，表示子进程的退出代码。
- child_stack：传入子进程使用的栈空间，也就是把用户态堆栈指针赋给子进程的 esp 寄存器。调用进程(指调用 clone() 的进程)应该总是为子进程分配新的堆栈。
- flags：表示使用哪些 CLONE_ 开头的标志位，与 namespace 相关的有CLONE_NEWIPC、CLONE_NEWNET、CLONE_NEWNS、CLONE_NEWPID、CLONE_NEWUSER、CLONE_NEWUTS 和 CLONE_NEWCGROUP。
- arg：指向传递给 fn() 函数的参数。

#### clone namespace示例(go)

go中提供了结合exec包和syscall进行系统调用：
```go
func main() {
        cmd := exec.Command("sh")
	//运行一个程序，返回一个Cmd*，用于对参数执行的程序进行设置，方便后面再run，返回的Cmd只设定了Path和Args两个参数
        cmd.SysProcAttr = &syscall.SysProcAttr{
		//保管可选的，各操作系统特定的sys执行属性
                Cloneflags: syscall.CLONE_NEWUTS,
		//linux下的clone calls,这里通过系统调,CLONE_NEWUTS，创建一个UTS Namespace
        }
        cmd.Stdin = os.Stdin
        cmd.Stdout = os.Stdout
        cmd.Stderr = os.Stderr
	//指定进程的Stdin,Stdout,Stderr，为os.Stdin而os.Stdin是指向了标准输入的文件描述符,对于linux是/dev/stdin文件
        if err := cmd.Run(); err != nil { //执行cmd设置的程序
                fmt.Println(err)
        }
}
```

可选的namespace:

	CLONE_NEWIPC                     = 0x8000000
	CLONE_NEWNET                     = 0x40000000
	CLONE_NEWNS                      = 0x20000
	CLONE_NEWPID                     = 0x20000000
	CLONE_NEWUSER                    = 0x10000000
	CLONE_NEWUTS                     = 0x4000000

#### linux fork、clone、vfork的区别

这三个API的内部实际都是调用一个内核内部函数do_fork，只是填写的参数不同而已。

vfork，其实就是fork的部分过程，用以简化并提高效率。而fork与clone是区别的。fork是进程资源的完全复制，包括进程的PCB、线程的系统堆栈、进程的用户空间、进程打开的设备等。而在clone中其实只有前两项是被复制了的，后两项都与父进程共享。

在四项资源的复制中，用户空间是相对庞大的，如果完全复制则效率会很低。在Linux中采用的“写时复制”技术，也就是说，fork执行时并不真正复制用户空间的所有页面，而只是复制页面表。这样，无论父进程还是子进程，当发生用户空间的写操作时，都会引发“写复制”操作，而另行分配一块可用的用户空间，使其完全独立。这是一种提高效率的非常有效的方法。

而对于clone来说，它们连这些页面表都是与父进程共享，故而是真正意义上的共享，因此对共享数据的保护必须有上层应用来保证。

### setns() 函数

通过 setns() 函数可以将当前进程加入到已有的 namespace 中。setns() 在 C 语言库中的声明如下：

	#define _GNU_SOURCE #include <sched.h> int setns(int fd, int nstype);

和 clone() 函数一样，C 语言库中的 setns() 函数也是对 setns() 系统调用的封装：

- fd：表示要加入 namespace 的文件描述符。它是一个指向 /proc/[pid]/ns 目录中文件的文件描述符，可以通过直接打开该目录下的链接文件或者打开一个挂载了该目录下链接文件的文件得到。
- nstype：参数 nstype 让调用者可以检查 fd 指向的 namespace 类型是否符合实际要求。若把该参数设置为 0 表示不检查。

前面我们提到：可以通过挂载的方式把 namespace 保留下来。保留 namespace 的目的是为以后把进程加入这个 namespace 做准备。

在 docker 中，使用 docker exec 命令在已经运行着的容器中执行新的命令就需要用到 setns() 函数。为了把新加入的 namespace 利用起来，还需要引入 execve() 系列的函数，该函数可以执行用户的命令，比较常见的用法是调用 /bin/bash 并接受参数运行起一个 shell。

### unshare() 函数 和 unshare 命令
通过 unshare 函数可以在原进程上进行 namespace 隔离。也就是创建并加入新的 namespace 。unshare() 在 C 语言库中的声明如下：

	#define _GNU_SOURCE
	#include <sched.h>
	int unshare(int flags);

和前面两个函数一样，C 语言库中的 unshare() 函数也是对 unshare() 系统调用的封装。调用 unshare() 的主要作用就是：不启动新的进程就可以起到资源隔离的效果，相当于跳出原先的 namespace 进行操作。

系统还默认提供了一个叫 unshare 的命令，其实就是在调用 unshare() 系统调用。下面的 demo 使用 unshare 命令把当前进程的 user namespace 设置成了 root：

	unsgare --map-root-user --user -sh -c whoami
![](https://oscimg.oschina.net/oscnet/up-e5a5d8950ea418cec540471918844d2fa8b.png)

# 总结

namespace 是 linux 内核提供的特性，为虚拟化而生。

随着 docker 的诞生引爆了容器技术，也把长期在后台默默奉献的 namespace 技术推到了大家的面前。

