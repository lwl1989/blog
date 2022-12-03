CreateTime:2021-10-11 17:56:34.0

> 引子：在之前的文章里 [golang netpoll的实现与分析](https://my.oschina.net/lwl1989/blog/5267119) 讲了一些，对于golang netpoll的实现，但是，数据是怎么通过硬件到达golang的这块不是太明确，今天就主要分析下这一块。

# linux的网络的基本实现
在 TCP/IP ⽹络分层模型⾥，整个协议栈被分成了物理层、链路层、⽹络层，传输层和应⽤层。物理层对应的是⽹卡和⽹线，应⽤层对应的是我们常⻅的 Nginx，FTP 等等各种应⽤。Linux 实现的是链路层、⽹络层和传输层这三层。

在 Linux 内核实现中，链路层协议靠⽹卡驱动来实现，内核协议栈来实现⽹络层和传输层。内核对更上层的应⽤层提供 socket 接⼝来供⽤户进程访问。我们⽤ Linux 的视⻆来看到的 TCP/IP ⽹络分层模型应该是下⾯这个样⼦的。

![](https://oscimg.oschina.net/oscnet/up-55a4932033cc181de1a8bc130e3cc69c7d3.png)

## 如何接收网络事件

当设备上有数据到达的时候，会给 CPU 的相关引脚上触发⼀个电压变化，以通知 CPU 来处理数据。

也可以把这个叫  **硬中断**

但是我们知道，cpu运行速度很快，但是网络读取数据会很慢，这时候就会长期占用cpu,导致cpu无法处理其他事件，比如，鼠标移动。

那么在linux中是怎么解决掉这个问题的呢？

linux内核将中断处理拆分开，拆分为了2个部分，一个是上面提到的 **硬中断**，另外就是 **软中断**。

第一部分接收到cpu电压变化，产生硬中断，然后只做最简单的处理，然后异步的交给硬件去接收信息到缓冲区。这个时候，cpu就已经可以接收其他中断信息过来了。

第二部分就是软中断部分，软中断是怎么做的呢？其实就是对内存的二进制位进行变更，类似于我们平常写业务常用的到的status字段一样，比如网络Io中，当缓冲区接收数据完毕，会将当前状态改为完成。举个例子，epoll读取某个io时间读取完数据时，并不会直接进入就绪态，而是等下次循环遍历判断状态，才会将这个fd塞入就绪列表（当然，这个时间很短，不过相对于cpu来说，这个时间就很长了）。

2.4 以后的内核版本采⽤的下半部实现⽅式是软中断，由 ksoftirqd 内核线程全权处理。和硬中断不同的是，硬中断是通过给 CPU 物理引脚施加电压变化，⽽软中断是通过给内存中的⼀个变量的⼆进制值以通知软中断处理程序。

这也就是为什么知道2.6才有epoll（正式引入）使用的原因，2.4以前内核都不支持这种方式。

总体的数据流转图如下：

![](https://oscimg.oschina.net/oscnet/up-d643a2970feb555d2cc80c816f5e1ab60aa.png)

一个数据从到达网卡，要经历以下步骤才会完成一次数据接收：

- 数据包从外面的网络进入物理网卡。如果目的地址不是该网卡，且该网卡没有开启混杂模式，该包会被网卡丢弃。
- 网卡将数据包通过DMA的方式写入到指定的内存地址，该地址由网卡驱动分配并初始化。注： 老的网卡可能不支持DMA，不过新的网卡一般都支持。
- 网卡通过硬件中断（IRQ）通知CPU，告诉它有数据来了
- CPU根据中断表，调用已经注册的中断函数，这个中断函数会调到驱动程序（NIC Driver）中相应的函数
- 驱动先禁用网卡的中断，表示驱动程序已经知道内存中有数据了，告诉网卡下次再收到数据包直接写内存就可以了，不要再通知CPU了，这样可以提高效率，避免CPU不停的被中断。
- 启动软中断。这步结束后，硬件中断处理函数就结束返回了。由于硬中断处理程序执行的过程中不能被中断，所以如果它执行时间过长，会导致CPU没法响应其它硬件的中断，于是内核引入软中断，这样可以将硬中断处理函数中耗时的部分移到软中断处理函数里面来慢慢处理。
- 内核中的ksoftirqd进程专门负责软中断的处理，当它收到软中断后，就会调用相应软中断所对应的处理函数，对于上面第6步中是网卡驱动模块抛出的软中断，ksoftirqd会调用网络模块的net_rx_action函数
- net_rx_action调用网卡驱动里的poll函数来一个一个的处理数据包
- 在pool函数中，驱动会一个接一个的读取网卡写到内存中的数据包，内存中数据包的格式只有驱动知道
- 驱动程序将内存中的数据包转换成内核网络模块能识别的skb格式，然后调用napi_gro_receive函数
- napi_gro_receive会处理GRO相关的内容，也就是将可以合并的数据包进行合并，这样就只需要调用一次协议栈。然后判断是否开启了RPS，如果开启了，将会调用enqueue_to_backlog
- 在enqueue_to_backlog函数中，会将数据包放入CPU的softnet_data结构体的input_pkt_queue中，然后返回，如果input_pkt_queue满了的话，该数据包将会被丢弃，queue的大小可以通过net.core.netdev_max_backlog来配置
- CPU会接着在自己的软中断上下文中处理自己input_pkt_queue里的网络数据（调用__netif_receive_skb_core）
- 如果没开启RPS，napi_gro_receive会直接调用__netif_receive_skb_core
- 看是不是有AF_PACKET类型的socket（也就是我们常说的原始套接字），如果有的话，拷贝一份数据给它。tcpdump抓包就是抓的这里的包。
- 调用协议栈相应的函数，将数据包交给协议栈处理。
- 待内存中的所有数据包被处理完成后（即poll函数执行完成），启用网卡的硬中断，这样下次网卡再收到数据的时候就会通知CPU

##  epoll

![](https://oscimg.oschina.net/oscnet/up-6a8bbc62ff2ffe87918f0db4619e982c417.png)

#### poll函数

这里的poll函数是说注册的回调函数，在软中断中进行处理的。比如epoll程序，会注册一个“ep_poll_callback”

以go epoll为例：

go:  accept –> pollDesc.Init -> poll_runtime_pollOpen –> runtime.netpollopen(epoll_create) -> epollctl(EPOLL_CTL_ADD)

go:  netpollblock（gopark）,让出cpu->调度回来，netpoll(0)将协程写入就绪态->其他操作......

epoll thread: epoll_create(ep_ptable_queue_proc,注册软中断到ksoftirqd，将方法ep_poll_callback注册到)->epoll_add->epoll_wait(ep_poll让出cpu)

core: 网卡接收到数据->dma+硬中断->软中断->系统调度到ksoftirqd，处理ep_poll_callback（这里要注意，新的连接进入到程序，不是通过callback,而是走accept）->获取到之前注册的fd句柄->copy网卡数据到句柄->根据事件类型，对fd进行操作（就绪列表）

![](https://oscimg.oschina.net/oscnet/up-8286de44293d5d076537e9811077a5e25f2.png)

###### 部分代码
go: accept
```
// accept阻塞，等待系统事件（等待有客户端进来）
func (fd *FD) Accept() (int, syscall.Sockaddr, string, error) {
	if err := fd.readLock(); err != nil {
		return -1, nil, "", err
	}
	defer fd.readUnlock()

	if err := fd.pd.prepareRead(fd.isFile); err != nil {
		return -1, nil, "", err
	}
	for {
		s, rsa, errcall, err := accept(fd.Sysfd)
		if err == nil {
			return s, rsa, "", err
		}
		switch err {
		case syscall.EAGAIN:
			if fd.pd.pollable() {
				if err = fd.pd.waitRead(fd.isFile); err == nil {
					continue
				}
			}
		case syscall.ECONNABORTED:
			// This means that a socket on the listen
			// queue was closed before we Accept()ed it;
			// it's a silly error, so try again.
			continue
		}
		return -1, nil, errcall, err
	}
}
//accept创建netpoll
func (fd *netFD) accept() (netfd *netFD, err error) {
	d, rsa, errcall, err := fd.pfd.Accept()
	if err != nil {
		if errcall != "" {
			err = wrapSyscallError(errcall, err)
		}
		return nil, err
	}

	if netfd, err = newFD(d, fd.family, fd.sotype, fd.net); err != nil {
		poll.CloseFunc(d)
		return nil, err
	}
	if err = netfd.init(); err != nil {  //open 创建 ctl_add
		fd.Close()
		return nil, err
	}
	lsa, _ := syscall.Getsockname(netfd.pfd.Sysfd)
	netfd.setAddr(netfd.addrFunc()(lsa), netfd.addrFunc()(rsa))
	return netfd, nil
}

### //syscall包。 最终调用的是linux的accept
func accept(s int, rsa *RawSockaddrAny, addrlen *_Socklen) (fd int, err error) {
	r0, _, e1 := syscall(funcPC(libc_accept_trampoline), uintptr(s), uintptr(unsafe.Pointer(rsa)), uintptr(unsafe.Pointer(addrlen)))
	fd = int(r0)
	if e1 != 0 {
		err = errnoErr(e1)
	}
	return
}

```

# epoll源码
```
 static int __init eventpoll_init(void)
{
   mutex_init(&pmutex);
   ep_poll_safewake_init(&psw);
   epi_cache = kmem_cache_create("eventpoll_epi", sizeof(struct epitem), 0, SLAB_HWCACHE_ALIGN|EPI_SLAB_DEBUG|SLAB_PANIC, NULL);
   pwq_cache = kmem_cache_create("eventpoll_pwq", sizeof(struct eppoll_entry), 0, EPI_SLAB_DEBUG|SLAB_PANIC, NULL);
   return 0;
}
```
### 基础数据结构

epoll用kmem_cache_create（slab分配器）分配内存用来存放struct epitem和struct eppoll_entry。
当向系统中添加一个fd时，就创建一个epitem结构体，这是内核管理epoll的基本数据结构：

	struct epitem {
		struct rb_node  rbn;        //用于主结构管理的红黑树
		struct list_head  rdllink;  //事件就绪队列
		struct epitem  *next;       //用于主结构体中的链表
		struct epoll_filefd  ffd;   //这个结构体对应的被监听的文件描述符信息
		int  nwait;                 //poll操作中事件的个数
		struct list_head  pwqlist;  //双向链表，保存着被监视文件的等待队列，功能类似于select/poll中的poll_table
		struct eventpoll  *ep;      //该项属于哪个主结构体（多个epitm从属于一个eventpoll）
		struct list_head  fllink;   //双向链表，用来链接被监视的文件描述符对应的struct file。因为file里有f_ep_link,用来保存所有监视这个文件的epoll节点
		struct epoll_event  event;  //注册的感兴趣的事件,也就是用户空间的epoll_event
	}

而每个epoll fd（epfd）对应的主要数据结构为：

	struct eventpoll {
		spin_lock_t       lock;        //对本数据结构的访问
		struct mutex      mtx;         //防止使用时被删除
		wait_queue_head_t     wq;      //sys_epoll_wait() 使用的等待队列
		wait_queue_head_t   poll_wait;       //file->poll()使用的等待队列
		struct list_head    rdllist;        //事件满足条件的链表
		struct rb_root      rbr;            //用于管理所有fd的红黑树（树根）
		struct epitem      *ovflist;       //将事件到达的fd进行链接起来发送至用户空间
	}

来张更细节的图：

![](https://oscimg.oschina.net/oscnet/up-a2e2b9e9e5623119f362313313c4952d39c.png)

struct eventpoll在epoll_create时创建。

```
long sys_epoll_create(int size) {

    struct eventpoll *ep;

   // ...

    ep_alloc(&ep); //为ep分配内存并进行初始化

/* 调用anon_inode_getfd 新建一个file instance，

也就是epoll可以看成一个文件（匿名文件）。

因此我们可以看到epoll_create会返回一个fd。

           epoll所管理的所有的fd都是放在一个大的结构eventpoll(红黑树)中，

将主结构体struct eventpoll *ep放入file->private项中进行保存（sys_epoll_ctl会取用）*/

 fd = anon_inode_getfd("[eventpoll]", &eventpoll_fops, ep, O_RDWR | (flags & O_CLOEXEC));
     return fd;
}
```
其中，ep_alloc(struct eventpoll **pep)为pep分配内存，并初始化。
其中，上面注册的操作eventpoll_fops定义如下：
`static const struct file_operations eventpoll_fops = {
    .release=  ep_eventpoll_release,
    .poll    =  ep_eventpoll_poll,
};
`
这样说来，内核中维护了一棵红黑树，大致的结构如下：
clip_image002
接着是epoll_ctl函数（省略了出错检查等代码）：
```
 asmlinkage long sys_epoll_ctl(int epfd,int op,int fd,struct epoll_event __user *event) {
    int error;
    struct file *file,*tfile;
    struct eventpoll *ep;
    struct epoll_event epds;
    error = -FAULT;
    //判断参数的合法性，将 __user *event 复制给 epds。
    if(ep_op_has_event(op) && copy_from_user(&epds,event,sizeof(struct epoll_event)))
            goto error_return; //省略跳转到的代码
    file  = fget (epfd); // epoll fd 对应的文件对象
    tfile = fget(fd);    // fd 对应的文件对象
    //在create时存入进去的（anon_inode_getfd），现在取用。
    ep = file->private->data;
    mutex_lock(&ep->mtx);
    //防止重复添加（在ep的红黑树中查找是否已经存在这个fd）
    epi = epi_find(ep,tfile,fd);
    switch(op)
    {
        case EPOLL_CTL_ADD:  //增加监听一个fd
            if(!epi)
            {
                epds.events |= EPOLLERR | POLLHUP;     //默认包含POLLERR和POLLHUP事件
                error = ep_insert(ep,&epds,tfile,fd);  //在ep的红黑树中插入这个fd对应的epitm结构体。
            } else  //重复添加（在ep的红黑树中查找已经存在这个fd）。
                error = -EEXIST;
            break;
        ...
    }
    return error;
}
```
### ep_insert的实现如下：
```
static int ep_insert(struct eventpoll *ep, struct epoll_event *event, struct file *tfile, int fd)
{
   int error ,revents,pwake = 0;
   unsigned long flags ;
   struct epitem *epi;
   /*
      struct ep_queue{
         poll_table pt;
         struct epitem *epi;
      }   */
   struct ep_pqueue epq;
   //分配一个epitem结构体来保存每个加入的fd
   if(!(epi = kmem_cache_alloc(epi_cache,GFP_KERNEL)))
      goto error_return;
   //初始化该结构体
   ep_rb_initnode(&epi->rbn);
   INIT_LIST_HEAD(&epi->rdllink);
   INIT_LIST_HEAD(&epi->fllink);
   INIT_LIST_HEAD(&epi->pwqlist);
   epi->ep = ep;
   ep_set_ffd(&epi->ffd,tfile,fd);
   epi->event = *event;
   epi->nwait = 0;
   epi->next = EP_UNACTIVE_PTR;
   epq.epi = epi;
   //安装poll回调函数
   init_poll_funcptr(&epq.pt, ep_ptable_queue_proc );
   /* 调用poll函数来获取当前事件位，其实是利用它来调用注册函数ep_ptable_queue_proc（poll_wait中调用）。
       如果fd是套接字，f_op为socket_file_ops，poll函数是
       sock_poll()。如果是TCP套接字的话，进而会调用
       到tcp_poll()函数。此处调用poll函数查看当前
       文件描述符的状态，存储在revents中。
       在poll的处理函数(tcp_poll())中，会调用sock_poll_wait()，
       在sock_poll_wait()中会调用到epq.pt.qproc指向的函数，
       也就是ep_ptable_queue_proc()。  */
   revents = tfile->f_op->poll(tfile, &epq.pt);
   spin_lock(&tfile->f_ep_lock);
   list_add_tail(&epi->fllink,&tfile->f_ep_lilnks);
   spin_unlock(&tfile->f_ep_lock);
   ep_rbtree_insert(ep,epi); //将该epi插入到ep的红黑树中
   spin_lock_irqsave(&ep->lock,flags);
//  revents & event->events：刚才fop->poll的返回值中标识的事件有用户event关心的事件发生。
// !ep_is_linked(&epi->rdllink)：epi的ready队列中有数据。ep_is_linked用于判断队列是否为空。
/*  如果要监视的文件状态已经就绪并且还没有加入到就绪队列中,则将当前的
    epitem加入到就绪队列中.如果有进程正在等待该文件的状态就绪,则
    唤醒一个等待的进程。  */
if((revents & event->events) && !ep_is_linked(&epi->rdllink)) {
      list_add_tail(&epi->rdllink,&ep->rdllist); //将当前epi插入到ep->ready队列中。
/* 如果有进程正在等待文件的状态就绪，
也就是调用epoll_wait睡眠的进程正在等待，
则唤醒一个等待进程。
waitqueue_active(q) 等待队列q中有等待的进程返回1，否则返回0。
*/
      if(waitqueue_active(&ep->wq))
         __wake_up_locked(&ep->wq,TAKS_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE);
/*  如果有进程等待eventpoll文件本身（???）的事件就绪，
           则增加临时变量pwake的值，pwake的值不为0时，
           在释放lock后，会唤醒等待进程。 */ 
      if(waitqueue_active(&ep->poll_wait))
         pwake++;
   }
   spin_unlock_irqrestore(&ep->lock,flags);
if(pwake)
      ep_poll_safewake(&psw,&ep->poll_wait);//唤醒等待eventpoll文件状态就绪的进程
   return 0;
}
```
	init_poll_funcptr(&epq.pt, ep_ptable_queue_proc);
	revents = tfile->f_op->poll(tfile, &epq.pt);
这两个函数将ep_ptable_queue_proc注册到epq.pt中的qproc。
	typedef struct poll_table_struct {
		poll_queue_proc qproc;
		unsigned long key;
	}poll_table;
执行f_op->poll(tfile, &epq.pt)时，XXX_poll(tfile, &epq.pt)函数会执行poll_wait()，poll_wait()会调用epq.pt.qproc函数，即ep_ptable_queue_proc。
ep_ptable_queue_proc函数如下：
```
/*  在文件操作中的poll函数中调用，将epoll的回调函数加入到目标文件的唤醒队列中。
    如果监视的文件是套接字，参数whead则是sock结构的sk_sleep成员的地址。  */
static void ep_ptable_queue_proc(struct file *file, wait_queue_head_t *whead, poll_table *pt) {
/* struct ep_queue{
         poll_table pt;
         struct epitem *epi;
      } */
    struct epitem *epi = ep_item_from_epqueue(pt); //pt获取struct ep_queue的epi字段。
    struct eppoll_entry *pwq;
    if (epi->nwait >= 0 && (pwq = kmem_cache_alloc(pwq_cache, GFP_KERNEL))) {
        init_waitqueue_func_entry(&pwq->wait, ep_poll_callback);
        pwq->whead = whead;
        pwq->base = epi;
        add_wait_queue(whead, &pwq->wait);
        list_add_tail(&pwq->llink, &epi->pwqlist);
        epi->nwait++;
    } else {
        /* We have to signal that an error occurred */
        /*
         * 如果分配内存失败，则将nwait置为-1，表示
         * 发生错误，即内存分配失败，或者已发生错误
         */
        epi->nwait = -1;
    }
}
```
### ep_ptable_queue_proc
其中struct eppoll_entry定义如下：
```
struct eppoll_entry {
   struct list_head llink;
   struct epitem *base;
   wait_queue_t wait;
   wait_queue_head_t *whead;
};
ep_ptable_queue_proc 函数完成 epitem 加入到特定文件的wait队列任务。
ep_ptable_queue_proc有三个参数：
struct file *file;              该fd对应的文件对象
wait_queue_head_t *whead;      该fd对应的设备等待队列（同select中的mydev->wait_address）
poll_table *pt;                 f_op->poll(tfile, &epq.pt)中的epq.pt
```
在ep_ptable_queue_proc函数中，引入了另外一个非常重要的数据结构eppoll_entry。eppoll_entry主要完成epitem和epitem事件发生时的callback（ep_poll_callback）函数之间的关联。首先将eppoll_entry的whead指向fd的设备等待队列（同select中的wait_address），然后初始化eppoll_entry的base变量指向epitem，最后通过add_wait_queue将epoll_entry挂载到fd的设备等待队列上。完成这个动作后，epoll_entry已经被挂载到fd的设备等待队列。

由于ep_ptable_queue_proc函数设置了等待队列的ep_poll_callback回调函数。所以在设备硬件数据到来时，硬件中断处理函数中会唤醒该等待队列上等待的进程时，会调用唤醒函数ep_poll_callback（参见博文http://www.cnblogs.com/apprentice89/archive/2013/05/09/3068274.html）。
```
static int ep_poll_callback(wait_queue_t *wait, unsigned mode, int sync, void *key) {
   int pwake = 0;
   unsigned long flags;
   struct epitem *epi = ep_item_from_wait(wait);
   struct eventpoll *ep = epi->ep;
   spin_lock_irqsave(&ep->lock, flags);
   //判断注册的感兴趣事件
//#define EP_PRIVATE_BITS  (EPOLLONESHOT | EPOLLET)
//有非EPOLLONESHONT或EPOLLET事件
   if (!(epi->event.events & ~EP_PRIVATE_BITS))
      goto out_unlock;
   if (unlikely(ep->ovflist != EP_UNACTIVE_PTR)) {
      if (epi->next == EP_UNACTIVE_PTR) {
         epi->next = ep->ovflist;
         ep->ovflist = epi;
      }
      goto out_unlock;
   }
   if (ep_is_linked(&epi->rdllink))
      goto is_linked;
    //***关键***，将该fd加入到epoll监听的就绪链表中
   list_add_tail(&epi->rdllink, &ep->rdllist);
   //唤醒调用epoll_wait()函数时睡眠的进程。用户层epoll_wait(...) 超时前返回。
if (waitqueue_active(&ep->wq))
      __wake_up_locked(&ep->wq, TASK_UNINTERRUPTIBLE | TASK_INTERRUPTIBLE);
   if (waitqueue_active(&ep->poll_wait))
      pwake++;
   out_unlock: spin_unlock_irqrestore(&ep->lock, flags);
   if (pwake)
      ep_poll_safewake(&psw, &ep->poll_wait);
   return 1;
}
```
所以ep_poll_callback函数主要的功能是将被监视文件的等待事件就绪时，将文件对应的epitem实例添加到就绪队列中，当用户调用epoll_wait()时，内核会将就绪队列中的事件报告给用户。


### epoll_wait实现如下：
```
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events, int, maxevents, int, timeout)  {
   int error;
   struct file *file;
   struct eventpoll *ep;
    /* 检查maxevents参数。 */
   if (maxevents <= 0 || maxevents > EP_MAX_EVENTS)
      return -EINVAL;
    /* 检查用户空间传入的events指向的内存是否可写。参见__range_not_ok()。 */
   if (!access_ok(VERIFY_WRITE, events, maxevents * sizeof(struct epoll_event))) {
      error = -EFAULT;
      goto error_return;
   }
    /* 获取epfd对应的eventpoll文件的file实例，file结构是在epoll_create中创建。 */
   error = -EBADF;
   file = fget(epfd);
   if (!file)
      goto error_return;
    /* 通过检查epfd对应的文件操作是不是eventpoll_fops 来判断epfd是否是一个eventpoll文件。如果不是则返回EINVAL错误。 */
   error = -EINVAL;
   if (!is_file_epoll(file))
      goto error_fput;
    /* At this point it is safe to assume that the "private_data" contains  */
   ep = file->private_data;
    /* Time to fish for events ... */
   error = ep_poll(ep, events, maxevents, timeout);
    error_fput:
   fput(file);
error_return:
   return error;
}
```
### epoll_wait调用ep_poll，ep_poll实现如下：
```
 static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout) {
    int res, eavail;
   unsigned long flags;
   long jtimeout;
   wait_queue_t wait;
    /* timeout是以毫秒为单位，这里是要转换为jiffies时间。这里加上999(即1000-1)，是为了向上取整。 */
   jtimeout = (timeout < 0 || timeout >= EP_MAX_MSTIMEO) ?MAX_SCHEDULE_TIMEOUT : (timeout * HZ + 999) / 1000;
 retry:
   spin_lock_irqsave(&ep->lock, flags);
    res = 0;
   if (list_empty(&ep->rdllist)) {
      /* 没有事件，所以需要睡眠。当有事件到来时，睡眠会被ep_poll_callback函数唤醒。*/
      init_waitqueue_entry(&wait, current); //将current进程放在wait这个等待队列中。
      wait.flags |= WQ_FLAG_EXCLUSIVE;
      /* 将当前进程加入到eventpoll的等待队列中，等待文件状态就绪或直到超时，或被信号中断。 */
      __add_wait_queue(&ep->wq, &wait);
       for (;;) {
         /* 执行ep_poll_callback()唤醒时应当需要将当前进程唤醒，所以当前进程状态应该为“可唤醒”TASK_INTERRUPTIBLE  */
         set_current_state(TASK_INTERRUPTIBLE);
         /* 如果就绪队列不为空，也就是说已经有文件的状态就绪或者超时，则退出循环。*/
         if (!list_empty(&ep->rdllist) || !jtimeout)
            break;
         /* 如果当前进程接收到信号，则退出循环，返回EINTR错误 */
         if (signal_pending(current)) {
            res = -EINTR;
            break;
         }
          spin_unlock_irqrestore(&ep->lock, flags);
         /* 主动让出处理器，等待ep_poll_callback()将当前进程唤醒或者超时,返回值是剩余的时间。
从这里开始当前进程会进入睡眠状态，直到某些文件的状态就绪或者超时。
当文件状态就绪时，eventpoll的回调函数ep_poll_callback()会唤醒在ep->wq指向的等待队列中的进程。*/
         jtimeout = schedule_timeout(jtimeout);
         spin_lock_irqsave(&ep->lock, flags);
      }
      __remove_wait_queue(&ep->wq, &wait);
       set_current_state(TASK_RUNNING);
   }
    /* ep->ovflist链表存储的向用户传递事件时暂存就绪的文件。
    * 所以不管是就绪队列ep->rdllist不为空，或者ep->ovflist不等于
    * EP_UNACTIVE_PTR，都有可能现在已经有文件的状态就绪。
    * ep->ovflist不等于EP_UNACTIVE_PTR有两种情况，一种是NULL，此时
    * 可能正在向用户传递事件，不一定就有文件状态就绪，
    * 一种情况时不为NULL，此时可以肯定有文件状态就绪，
    * 参见ep_send_events()。
    */
   eavail = !list_empty(&ep->rdllist) || ep->ovflist != EP_UNACTIVE_PTR;
    spin_unlock_irqrestore(&ep->lock, flags);
    /* Try to transfer events to user space. In case we get 0 events and there's still timeout left over, we go trying again in search of more luck. */
   /* 如果没有被信号中断，并且有事件就绪，但是没有获取到事件(有可能被其他进程获取到了)，并且没有超时，则跳转到retry标签处，重新等待文件状态就绪。 */
   if (!res && eavail && !(res = ep_send_events(ep, events, maxevents)) && jtimeout)
      goto retry;
    /* 返回获取到的事件的个数或者错误码 */
   return res;
}
```

# 小知识

#### 混杂模式

混杂模式（英语：promiscuous mode）是电脑网络中的术语。是指一台机器的网卡能够接收所有经过它的数据流，而不论其目的地址是否是它。

混杂模式常用于网络分析

#### DMA

DMA，全称Direct Memory Access，即直接存储器访问。

DMA传输将数据从一个地址空间复制到另一个地址空间，提供在外设和存储器之间或者存储器和存储器之间的高速数据传输。当CPU初始化这个传输动作，传输动作本身是由DMA控制器来实现和完成的。DMA传输方式无需CPU直接控制传输，也没有中断处理方式那样保留现场和恢复现场过程，通过硬件为RAM和IO设备开辟一条直接传输数据的通道，使得CPU的效率大大提高。

DMA的主要特征:
- 每个通道都直接连接专用的硬件DMA请求，每个通道都同样支持软件触发，这些功能通过软件来配置。
- 在同一个DMA模块上，多个请求间的优先权可以通过软件编程设置（共有四级：很高、高、中等和低），优先权设置相等时由硬件决定（请求0优先于请求1，依此类推）。
- 独立数据源和目标数据区的传输宽度（字节、半字、全字），模拟打包和拆包的过程。源和目标地址必须按数据传输宽度对齐。
- 支持循环的缓冲器管理。
- 每个通道都有3个事件标志（DMA半传输、DMA传输完成和DMA传输出错），这3个事件标志逻辑或成为一个单独的中断请求。
- 存储器和存储器间的传输、外设和存储器、存储器和外设之间的传输。
- 闪存、SRAM、外设的SRAM、APB1、APB2和AHB外设均可作为访问的源和目标。
- 可编程的数据传输数目：最大为65535（0xFFFF）。

#### 非阻塞socket编程处理EAGAIN错误

　　在linux进行非阻塞的socket接收数据时经常出现Resource temporarily unavailable，errno代码为11(EAGAIN)，这是什么意思？
　　这表明你在非阻塞模式下调用了阻塞操作，在该操作没有完成就返回这个错误，这个错误不会破坏socket的同步，不用管它，下次循环接着recv就可以。对非阻塞socket而言，EAGAIN不是一种错误。在VxWorks和Windows上，EAGAIN的名字叫做EWOULDBLOCK。

# 参考

[epoll源码](https://www.cnblogs.com/apprentice89/p/3234677.html)
