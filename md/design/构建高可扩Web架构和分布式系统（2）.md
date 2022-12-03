CreateTime:2015-10-10 03:51:29.0

<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>构建快速可伸缩的数据访问块</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在讨论完设计分布式系统的核心考虑因素后，下面让我们再一起讨论难点部分：可扩展的数据访问。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
大多数简单的Web应用程序，例如LAMP堆栈应用程序，看起来如图5所示：
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="349" height="104" src="http://static.oschina.net/uploads/img/201510/10035131_1gxJ.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
&nbsp;图5：简单的Web应用程序
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
随着系统渐渐扩大，他们将面临两大主要挑战：构建可扩展的应用程序服务器和数据访问机制。在一个高可扩的应用程序设计中，通常最小化的应用程序（或Web）服务往往能体现一种无共享的架构。这就使得应用程序服务器层要进行横向扩展，这种设计的结果就是把繁重的工作转移到堆栈下层的数据库服务和配置服务上，这才是这一层上真正的可扩展和性能挑战。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
本文的剩余部分专门讨论一些常见策略和方法来使这些服务可以快速和可扩展，提升数据的访问速度。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="271" height="96" src="http://static.oschina.net/uploads/img/201510/10035131_NuNM.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图6 过于简化的的Web应用程序
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
大多数系统可能会简化成图6那样，这是个非常不错的起点。如果你有大量的数据，想要快速便捷地访问，就好比在你书桌抽屉的最上面有一堆糖果。虽然过于简化，但也暗示了两个难题：可扩展存储和快速的数据访问。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
为了这个例子，让我们假设有许多太字节（TB）数据，并且允许用户随机访问一小部分数据（见图7）。这与本文图片应用程序里的在文件服务器上定位一个图片文件非常相似。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="686" height="233" src="http://static.oschina.net/uploads/img/201510/10035131_QlPm.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图7 访问特定数据
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
这也是个非常大的挑战，把TB级数据加载到内存中的成本比较昂贵，这可以直接转化到磁盘上进行IO。从磁盘上读取要比内存慢的多——内存访问和Chuck Norris一样快，而磁盘的访问速度要比在DMV上慢。这个速度不同于大数据集上的合计，实数内存访问大概要比顺序访问快6倍，或者比随机从磁盘上读取快10万倍（参考<a target="_blank" href="http://queue.acm.org/detail.cfm?id=1563874" rel="nofollow">The Pathologies of Big Data</a>）。此外，即使是unique ID，想要在较少的数据中查找确切的位置也是一项艰巨的任务。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
幸运的是，能找到许多方法让这个问题变的简单，这里提供4个非常重要的方法：缓存、代理、索引和负载均衡器。在下面将会详细讨论这4个内容来提升数据访问速度。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<span></span> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>缓存</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
缓存就是利用本地参考原则：当CPU要读取一个数据时，首先从缓存中查找，找到就立即读取并送给CPU处理；没有找到，就用相对慢的速率从内存中读取并送给CPU处理，同时把这个数据所在的数据块调入缓存中，可以使得以后对整块数据的读取都从缓存中进行，不必再调用内存。它们几乎被用在每一个计算层上：硬件、操作系统、Web浏览器、Web应用程序等。一个缓存就相当于是一个临时内存：它有一个有限的空间量，但访问它比访问原始数据速度要快。缓存也可以存在于各个层的架构中，但经常在离前端最近的那个层上发现，在那里可以快速实现并返回数据，无需占用下游层数据。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
那么如何在我们的API例子里利用缓存使数据访问更快呢？在这种情况下，有许多地方可以插入缓存。一种是在请求层节点上插入缓存，如图8所示。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="334" src="http://static.oschina.net/uploads/img/201510/10035223_9avh.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图8 在请求层节点插入缓存
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在请求层节点上放置一个缓存，即可响应本地的存储数据。当对服务器发送一个请求时，如果本地存在所请求数据，那么该节点即会快速返回本地缓存数据。如果本地不存在，那么请求节点将会查询磁盘上的数据。请求层节点缓存即可以存在于内存中（这个非常快速）也可以位于该节点的本地磁盘上（比访问网络存储要快）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
&nbsp;
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="438" height="443" src="http://static.oschina.net/uploads/img/201510/10035223_O2fd.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图9 多个缓存
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
当扩展到许多节点的时候，会发生什么呢？如图9所示，如果请求层被扩展为多个节点，它仍然有可能访问每个节点所在的主机缓存。然而，如果你的负载均衡器随机分布节点之间的请求，那么请求将会访问各个不同的节点，因此缓存遗漏将会增加。这里有两种方法可以克服这个问题：全局缓存和分布式缓存。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>全局缓存</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
顾名思义，全局缓存是指所有节点都使用同一个缓存空间。这包含添加一台服务器或某种类型的文件存储，所有请求层节点访问该存储要比原始存储快。每个请求节点会以同种方式查询缓存，这种缓存方案可能有点复杂，随着客户机和请求数量的增加，单个缓存（Cache）很容易溢出，但在某些结构中却是非常有效的（特别是那些特定的硬件，专门用来提升全局缓存速度，或者是需要被缓存的特定数据集）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在图10中描述了全局缓存常见的两种方式。当一个Cache响应在高速缓存中没有发现时，Cache自己会从底层存储中检索缺少的那块数据。如图11所示，请求节点去检索那些在高速缓存中没有发现的数据。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="299" src="http://static.oschina.net/uploads/img/201510/10035237_p1A5.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图10 负责检索数据的全局缓存
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="383" src="http://static.oschina.net/uploads/img/201510/10035237_tHpA.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图11 全局缓存里负责检索的请求节点
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
大多使用全局缓存的应用程序都倾向于使用第一种类型，利用Cache本身来驱逐和获取数据以防止来自客户端的同一个数据区发出大量的请求。然而，在某些情况下，使用第二种实现反而更有意义。例如，如果该缓存用于存储大量的文件，低缓存的命中率会导致高速缓冲区不堪重负和缓存遗漏，在这种情况下， it helps to have a large percentage of the total data set (or hot data set) in the cache.
</p>
<p>
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>分布式缓存</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
分布式缓存即缓存在分布式系统各节点内存中的缓存数据。如图12所示，每个节点都有自己的缓存数据，所以如果冰箱扮演着缓存杂货店的角色，那么分布式缓存就是把食物放置在不同的地方——冰箱、橱柜和饭盒——当索取的时候，方便拿哪个就拿哪个，而无需特地往商店跑一趟。通常情况下，会使用一致性哈希函数对缓存进行划分，例如，一个请求节点正在寻找一个特定块的数据，在分布式缓存中，它很快就会知道去哪里找，并确保这些数据是可用的。这种情况下，每个节点都会有一小块缓存，然后在向另一个节点发送数据请求。因此分布式缓存的优点之一就是只需向请求池添加节点即可增加缓存空间，减少对数据库的访问负载量。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
当然，分布式缓存也存在缺点，例如单点实效，当该节点出现故障或宕机，那么该节点保存的数据就会丢失。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="480" height="492" src="http://static.oschina.net/uploads/img/201510/10035255_I61K.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图12 分布式缓存
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
分布式缓存的突出优点是提高运行速度（前提当然是正确实现）。选择不同的方法也会有不一样的效果，如果方法正确，即使请求数很多，也不会对速度有所影响。然而，缓存的维护需要额外的存储空间，这些通常需要购买存储器实现，但价格都很昂贵。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
其中一个非常流行的开源缓存产品：<a target="_blank" href="http://memcached.org/" rel="nofollow">Memcached</a>（即可以在本地缓存上工作也可以在分布式缓存上工作）；然而，这里还有许多其他选项（包括许多语言——或者是框架——特定选项）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
Memcached用于许多大型Web站点，其非常强大。Memcached基于一个存储键/值对的hashmap，优化数据存储和实现快速搜索（O(1)）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
Facebook采用不同类型的缓存技术来提升他们的网站性能（参考“<a target="_blank" href="http://sizzo.org/talks/" rel="nofollow">Facebook caching and performance</a>”）。在语言层面上使用$GLOBALS和APC（在PHP里提供函数调用），这有助于使中间函数调用更快（大多数语言都使用这些类型库来提升网站页面性能）。Facebook使用全局缓存并且通过多台服务器对缓存进行分布（参考“<a target="_blank" href="http://www.facebook.com/note.php?note_id=39391378919" rel="nofollow">Scaling memcached at Facebook</a>”），这就允许他们通过配置用户文件数据来获得更好的性能和吞吐量，并且还可以有一个中心位置更新数据（这是非常重要的，当运行成千上万台服务器时，缓存实效和一致性维护都是非常大的挑战）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
下面让我们谈谈当数据不在缓存中时，我们该做什么……
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>代理</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
简单点讲，代理服务器就是硬件/软件的中间件，接受客户端请求并且将他们转发到后端的源服务器上。通常，代理服务器用于过滤请求、记录请求日志或有时转换请求（通过添加/删除头结点、加密/解密或压缩）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="397" height="92" src="http://static.oschina.net/uploads/img/201510/10035320_eRUu.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图13 代理服务器
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
代理可以优化请求，并且从整个系统的角度来优化请求通信量。一方面，使用代理可以加速数据访问，可以把相同（或相似的）请求重叠压缩成一个请求，然后返回单个结果到请求客户端，这就是压缩转发（collapsed forwarding）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在一个局域网代理中，例如，客户端无需使用它们自己的IP去连接互联网，对于相同的内容，局域网将压缩来自客户端的请求。它很容易造成混淆，因为很多代理同样也是缓存（它是一个非常合乎逻辑放置缓存的地方），但并非所有缓存都扮演代理这一角色。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="708" height="253" src="http://static.oschina.net/uploads/img/201510/10035320_XNsa.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图14 使用一个代理服务器压缩请求
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
使用代理服务器的另一伟大方式是通过压缩请求对空间数据进行加密。采用这种策略最大化数据本地化的请求会导致减少请求的延迟。例如这里有一大串的节点请求B：partB1、partB2等等。我们可以设置代理来识别个人空间的位置请求，把它们压缩成单一的请求并只返回bigB，大大减少了从数据源处读取数据次数（如图15所示）。在高负载的情况下，代理也特别有用，或者当你具有有限的缓存时，它们基本上可以把多个请求批处理成一个。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="792" height="253" src="http://static.oschina.net/uploads/img/201510/10035320_dJee.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图15 使用代理对空间数据请求进行压缩
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>索引</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
使用索引来快速访问和优化数据是一个众所周知的策略，最有名的莫过于数据库索引。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="480" height="285" src="http://static.oschina.net/uploads/img/201510/10035426_YUVh.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图16 索引
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
一个索引就是数据库表的目录，表中数据和相应的存储位置的列表。好比是一篇文章的目录，可以加快数据表的。例如让我们来查找一块数据，B中的第二部分——如何发现它的位置？如果你通过数据类型存储了一个索引——例如数据A、B、C——它将告诉你数据B的原始位置。然后你只需去查看B并且根据需要阅读B的数据即可（参考图16）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
这些索引通常存储在内存或者是传入客户端请求的本地中。Berkeley DBs（BDBs）和树数据结构常常被用在有序列表中存储数据，这是访问索引的理想选择。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
通常，会把许多层索引作为一个映射，从一个位置移到下一个，以此类推，直到你得到想要的特定块数据（参照图17）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="220" src="http://static.oschina.net/uploads/img/201510/10035426_L2Zs.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图17 多层索引
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
索引也可以对相同的数据创建多个不同的视图。对大型数据集来说，这种方法是非常好的，无需创建多个额外的数据副本就可以定义不同的过滤和排序，
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
例如，早期的图像托管系统实际上是托管图像书本内容，允许客户端查询这些图像中的内容，输入一个主题，就可以把所有相关的内容搜索出来。此外，采用同样的方式，搜索引擎还允许你搜索出HTML内容。在这种情况下，需要很多的服务器来存储这些文件，查找其中一个页面可能会很麻烦。首先，反向索引查询任意个单词或字元祖都需要可以轻松地访问；再有就是导航到正确的页面和位置，检索到正确的图像结果也是项挑战。因此，在这种情况下，反向索引会映射到一个位置（例如书B），然后书B可能会有一个包含所有内容、位置和各个部分出现次数的索引。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="268" height="92" src="http://static.oschina.net/uploads/img/201510/10035426_ovAW.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
这种中间级索引只包含了Words、位置和书B的信息。与所有的信息不得不存储到一个大的反向索引中相比，这种嵌套的索引架构允许每个索引占用较少的空间。在大型系统中，这是非常关键的，即使采用压缩，这些索引也需要占用相当昂贵的存储空间。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
例如，让我们假设这个世界上有——100,000,000本书（参考<a target="_blank" href="http://booksearch.blogspot.com/2010/08/books-of-world-stand-up-and-be-counted.html" rel="nofollow">Inside Google Books</a>官方博客）——每本书只有10页，每页只有250个单词，这也就意味着有2500亿个单词。如果每个单词只有5个字节，每个字节占用8 bits（或1个byte，甚至有些字符占用2 bytes），所以5 bytes/单词，那么一个索引所包含的单词就有可能超过一个TB的存储。此外，索引还有可能包含其他信息，例如元祖单词、数据位置等。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
能够快速、轻松地找到数据是非常重要的，而使用索引就可以简单高效的实现。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>负载均衡器</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
分布式系统的另一个关键部分是负载均衡。负载均衡器几乎是每个架构的主要组成部分，他们的角色是负责把网络请求分散到一个服务器集群中的可用服务器上去，通过管理进入的Web数据流量和增加有效的网络带宽，从而使网络访问者获得尽可能最佳的联网体验的硬件设备。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="483" height="206" src="http://static.oschina.net/uploads/img/201510/10035517_ZtaK.png"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图18 负载均衡器
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
这里有许多种算法可用于为请求提供服务，包括随机选择一个节点、循环或者甚至是基于某个特定的标准来选择节点，例如内存或CPU利用率。负载均衡器即可以以硬件的方式表现出来，也可以以软件的方式。<a target="_blank" href="http://haproxy.1wt.eu/" rel="nofollow">HAProxy</a>是一个开源的负载均衡器，并且得到了非常广泛的使用。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在一个分布式系统中，负载均衡器通常处于系统的前端位置，所有传入的请求会相应地被路由。在一个复杂的分布式系统中，一个请求被路由到多个负载均衡器上并不常见，如图19所示：
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="135" src="http://static.oschina.net/uploads/img/201510/10035517_UX4w.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图19 多个负载平衡器
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
和代理一样，有些负载均衡器也可以基于请求的类型路由到不同的服务器集群上。（技术上来讲，这也被称为反向代理。）
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<span style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;line-height:24px;background-color:#FFFFFF;">负载均衡器所面临的挑战之一是管理用户特有的会话（user-session-specific）数据。</span> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
在一个大型系统里会有各种不同类型的调度和负载均衡算法，包括简单点的像随机选择或循环以及更复杂的机制，例如利用率和容量。所有的这些算法都可以分布流量和请求，并且提供有用的可靠性工具，像自动故障转移或者自动清除一个坏的节点（例如当它无法响应时）。然而，这种高级功能会把问题诊断的复杂。 例如，当遇到高负载情况时，负载均衡器将会移除变慢或超时的节点（因为请求太多，删除节点后会把请求分配到其他节点上），这无疑会加剧其他节点的工作量，即负载加重。这种情况下，大量的监测变的非常重要，因为整个系统流量和吞吐量看起来可能会减少（因为节点服务更少的请求），但可能会累坏个别节点（处理更多的请求）。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<span style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;line-height:21px;background-color:#FFFFFF;">负载均衡器也是扩展系统容量的一种简单方式，像文中提到的其他技术，在分布式系统架构中发挥着非常重要的作用。<span style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;line-height:21px;background-color:#FFFFFF;">负</span><span style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;line-height:21px;background-color:#FFFFFF;">载均衡器也提供一些重要功能来查看节点的健康状况，例如，如果该节点响应迟钝或过载，它可能就会被删除，然后利用系统中不同的节点冗余。</span></span><span></span> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<br>
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<strong>队列</strong> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
到目前为止，我们已经讨论了许多方法来加快数据读取速度，但扩展数据层的另一个重要组成部分是如何高效的写入数据。在简单的系统中，进程负载等都比较少，并且数据库比较小，毋庸置疑，写的速度肯定不会慢。然而，在大型复杂的系统里，这个速度就很难把握了，可能会花费很长的时间。例如，数据有可能要写到几个不同的地方，不同的服务器或索引、或者系统正处于高负载情况下。在这种情况，该在哪里进行写？或者其他任何任务都有可能花费很长时间，要想在系统实现性能和可用性需要构建异步。处理这种异步的一种常见的方式就是采用队列。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="500" height="404" src="http://static.oschina.net/uploads/img/201510/10035731_ZsWS.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图20 同步请求
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
想象在一个系统里，每个客户机都要把请求发送至远程服务器，那么服务器应该尽可能快的接收并完成任务，然后把结果返回到相应的客户端。在小型系统中，一台服务器（或逻辑服务器）传入客户端数据会与客户端发出时一样快，这样就比较完美了。然而，当服务器接收到的请求多余它的处理能力时，那么每个客户端必须排队等待服务器处理其他客户端请求，直到轮到你了，服务器才会处理你的请求，直到最终完成。这就是一个同步请求的例子，如图20所示。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
这种同步行为会严重降低客户端性能，客户端被迫等待，而通过添加额外的服务器来满足负载并不能解决问题，即使采用最有效的负载均衡也很难保证分配公平，在大客户端下。进一步讲，如果处理请求的服务器不可用或者瘫痪，那么客户端上游也将失败。有效的解决这个问题需要抽象客户端请求以及服务请求的实际工作。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
<img alt="" width="480" height="296" src="http://static.oschina.net/uploads/img/201510/10035731_jlLn.jpg"> 
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;text-align:center;background-color:#FFFFFF;">
图21 使用队列来管理请求
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
队列就像听起来那样简单，一个任务进来，就添加到队列里去，然后the workers挑选有能力处理的下一个任务。（参考图21）这些任务有可能仅是简单的写入，也有可能是复杂的，如把文档生成图像预览。当一个客户端把任务请求提交的队列中时，他们不需要被迫等待结果，相反，他们只需确认请求是否被正确接收。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
队列使客户端能够以异步的方式工作，对客户端请求和响应提供战略抽象。另一方面，在一个同步系统中，请求和回应是没有分化的，因此他们不能被分开管理。在异步系统中，客户端发出请求任务，服务器对收到的消息进行响应并确认任务被接收，然后客户端可以定期检查任务状态，一旦任务完成，即可看到结果。当客户端在等待异步请求是否完成时，它还可以自由执行其他任务，甚至是向其他服务器发出异步请求。下面要介绍的是消息和队列在分布式系统中的杠杆作用。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
队列也对服务中断或失败提供一种保护机制。例如，它很容易创建一个高度健壮的队列，当服务器瞬间失败时，该队列可以把刚刚失败的请求重新发送至服务器。相比直接暴露客户端来间断服务供应，使用队列来保证服务质量更可取，要求必须有复杂且矛盾性的客户端差错处理。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
队列是管理分布式通信与任何大规模分布式系统中各个部分之间的基础，并且有许多实现方式。这里有许多开源的队列，如<a target="_blank" href="http://www.rabbitmq.com/" rel="nofollow">RabbitMQ</a>、<a target="_blank" href="http://activemq.apache.org/" rel="nofollow">ActiveMQ</a>、<a target="_blank" href="http://kr.github.com/beanstalkd/" rel="nofollow">BeanstalkD</a>，但也有一些当做服务使用，如<a target="_blank" href="http://zookeeper.apache.org/" rel="nofollow">Zookeeper</a>，甚至是用来数据存储，像<a target="_blank" href="http://redis.io/" rel="nofollow">Redis</a>。
</p>
<p style="color:#333333;font-family:Helvetica, Tahoma, Arial, sans-serif;font-size:14px;background-color:#FFFFFF;">
<span></span> 
</p>