CreateTime:2015-11-04 14:24:02.0

>CI3的Session的重大改变就是默认使用了原生的Session，这符合Session类库本来的意思，似乎更加合理一些。总体来说，虽然设计理念不同，但为了保证向后兼容性，类库的使用方法与CI2.0的差别不是很大。

###Session 驱动
CI3的Session 类自带了 4 种不同的驱动（或叫做存储引擎）可供使用：
- files
- database
- redis
- memcached

我以redis为session的存储方案

###错误出现
当我将SESSION进行存储的时候，发现跨页面时，session会丢失。

###定位错误日志
```
ERROR - 2015-11-04 13:29:34 --> Session: Error while trying to obtain lock for ci_session:26b7716c39e799d7c6eac1357bc3fdd676bb6979
```     

经过代码跟踪，发现错误在这里报出
```
if ( ! $this->_redis->setex($lock_key, 300, time()))
			{
				log_message('error', 'Session: Error while trying to obtain lock for '.$this->_key_prefix.$session_id);
				return FALSE;
			}
```


####ci官方的意思是

>attempts to obtain a lock, in case another request already has it
生成一个锁，防止被重复使用

####那么 setex是啥列？
redis官方给出的解释
将值 value 关联到 key ，并将 key 的生存时间设为 seconds (以秒为单位)。
如果 key 已经存在， SETEX 命令将覆写旧值。


那么这个东西怎么能执行失败呢？不懂。
所以我进入命令行redis-cli

执行：
```
setex ci_session:26b7716c39e799d7c6eac1357bc3fdd676bb6979:lock 300 ddasda
```
报错：
```
(error) MISCONF Redis is configured to save RDB snapshots, but is currently not able to persist on disk. Commands that may modify the data set are disabled. Please check Redis logs for details about the error.
```
无法被保存到磁盘上
网上的解决方案是：
    在/etc/sysctl.conf 添加一项 'vm.overcommit_memory = 1' ，然后重启（或者运行命令'sysctl vm.overcommit_memory=1' ）使其生效。


网上查了一下，有人也遇到类似的问题，并且给出了很好的分析（详见：http://www.linuxidc.com/Linux/2012-07/66079.htm），简单地说：Redis在保存数据到硬盘时为了避免主进程假死，需要Fork一份主进程，然后在Fork进程内完成数据保存到硬盘的操作，如果主进程使用了4GB的内存，Fork子进程的时候需要额外的4GB，此时内存就不够了，Fork失败，进而数据保存硬盘也失败了。



###最终解决方案：
再redie.conf里，将stop-writes-on-bgsave-error配置的yes=>no
















