CreateTime:2015-10-05 15:20:05.0

<div>
<p>
分布式文件系统比较出名的有HDFS 和 GFS，其中HDFS比较简单一点。本文是一篇描述非常简洁易懂的漫画形式讲解HDFS的原理。比一般PPT要通俗易懂很多。不难得的学习资料。
</p>
<p>
1、三个部分: 客户端、nameserver（可理解为主控和文件索引类似linux的inode）、datanode（存放实际数据的存server）
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275491EUuz.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152002_Qf0T.png" width="1254" height="330"></a> 
</p>
<p>
&nbsp;
</p>
<p>
2、如何写数据过程
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275498Z1oo.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152002_xme9.png" width="1239" height="691"></a> 
</p>
<p>
&nbsp;
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275507R78E.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152003_ImMh.png" width="1245" height="700"></a> 
</p>
<p>
&nbsp;
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275516IB54.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152003_ObcQ.png" width="1244" height="318"></a> 
</p>
<p>
3、读取数据过程
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275526ud42.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152003_4m4l.png" width="1239" height="707"></a> 
</p>
<p>
4、容错：第一部分：故障类型及其检测方法（nodeserver 故障，和网络故障，和脏数据问题）
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275539Te06.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152004_5SYN.png" width="1233" height="692"></a> 
</p>
<p>
&nbsp;
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_13422755514VTl.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152004_cWfe.png" width="1242" height="652"></a> 
</p>
<p>
5、容错第二部分：读写容错
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_13422755647m0M.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152005_0MeR.png" width="1250" height="692"></a> 
</p>
<p>
6、容错第三部分：dataNode 失效
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_13422755744TT0.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152005_Hxgq.png" width="1236" height="683"></a> 
</p>
<p>
7、备份规则
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_13422755887K2V.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152005_sJGw.png" width="1249" height="690"></a> 
</p>
<p>
8、结束语
</p>
<p>
<a href="http://blog.chinaunix.net/attachment/201207/14/27105712_1342275600Tfg3.png" target="_blank" rel="nofollow"><img title="image" alt="image" src="http://static.oschina.net/uploads/img/201510/05152006_JDga.png" width="1234" height="376"></a> 
</p>
</div>
<div>
<div>
<div>
<a href="http://blog.chinaunix.net/uid-27105712-id-3274395.html#" rel="nofollow"></a><a href="http://blog.chinaunix.net/uid-27105712-id-3274395.html#" rel="nofollow"></a><a href="http://blog.chinaunix.net/uid-27105712-id-3274395.html#" rel="nofollow"></a><a href="http://blog.chinaunix.net/uid-27105712-id-3274395.html#" rel="nofollow"></a><a href="http://blog.chinaunix.net/uid-27105712-id-3274395.html#" rel="nofollow"></a> 
</div>
</div>
</div>