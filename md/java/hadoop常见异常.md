CreateTime:2015-10-08 10:39:05.0


###1、Shell$ExitCodeException
现象：运行hadoop job时出现如下异常：
```
Exception from container-launch: org.apache.hadoop.util.Shell$ExitCode
Exception: 
Container exited with a non-zero exit code 1
```
原因及解决办法：原因未知。重启可恢复正常
###2、Safe mode 
现象：分配map reduce任务时产生：
```
org.apache.hadoop.dfs.SafeModeException: Cannot delete /user/hadoop/input. Name node is in safe mode
```
说明Hadoop的NameNode处在安全模式下。 

经查阅：
安全模式主要是为了系统启动的时候检查各个DataNode上数据块的有效性，同时根据策略必要的复制或者删除部分数据块。

在分布式文件系统启动的时候，开始的时候会有安全模式，当分布式文件系统处于安全模式的情况下，文件系统中的内容不允许修改也不允许删除，直到安全模式结束。

用户可以通过dfsadmin -safemode value 来操作安全模式，参数value的说明如下：
enter - 进入安全模式
leave - 强制NameNode离开安全模式
get - 返回安全模式是否开启的信息
wait - 等待，一直到安全模式结束。

解决方案：
hadoop dfsadmin -safemode leave   （离开安全模式）

###3、超时错误 SocketTimeoutException
现象：
```
java.net.SocketTimeoutException: 66000 millis timeout while waiting for channel to be ready for read. c
```

产生原因：
        1.master和slave时钟不同步
           由于hadoop集群中的心跳和反馈机制，所以在配置的时候我们会进行时钟同步的操作，当某种原因造成cmos断电后，时钟会错乱，这个时间会产生此异常。
       解决方案：
           手动设置timezone   同步时间      参考命令  ntpdate  date等命令
    

     2.  由于网络卡顿引起   
       解决方案：设置hadoop集群的修改超时设置
         

在配置文件中，加入超时设置
```
<property>
<name>dfs.datanode.socket.write.timeout</name>
<value>3000000</value>
</property>

<property>
<name>dfs.socket.timeout</name>
<value>3000000</value>
</property>

```


###4、Permission denied: user=li, access=WRITE, inode="":zkpk:supergroup:rwxr-xr-x
原因为用户权限不足，不能访写HDFS中的文件。
解决方案1：
关闭hadoop权限，在hdfs-site.xml文件中添加
```
<property>    
<name>dfs.permissions</name>    
<value>false</value>    
</property>
```
解决方案2：
设置权限

###5、could only be replicated to 0 nodes, instead of 1解决办法

现象：
```
hadoop fs -put /home/hadoop/file/* input
java.io.IOException: File /user/hadoop/input/file1.txt could only be replicated to 0 nodes, instead of 1
```
产生原因：
1、系统或hdfs是否有足够空间(本人就是因为硬盘空间不足导致异常发生)
2、datanode数是否正常
3、是否在safemode
4、防火墙是否关闭
5、关闭hadoop、格式化、重启hadoop



如果put时出现java.io.IOException: Not a file:  hdfs://localhost:9000/user/icymary/input/test-in 
解决办法是hadoop dfs -rmr input
hadoop dfs -put /home/test-in input
原因是，当执行了多次put之后，就会在分布式文件系统中生成子目录，删除重新put即可。
如果在 hadoop jar hadoop-0.16.0-examples.jar wordcount input output该过程中出现"can only be replicated to node 0, instead of 1"，解决办法是，给磁盘释放更多的空间
如果 bin/hadoop jar hadoop-0.16.0-examples.jar wordcount input output过程中
INFO mapred.JobClient: map 0% reduce 0%

且一直卡住，在log日志中也没有出现异样，那么解决办法是，把/etc/hosts里面多余的机器名删掉，即可。







