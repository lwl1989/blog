CreateTime:2018-09-27 10:26:50.0

> 由于本人画图太渣，本文图片均copy自互联网

## 什么是I/O

通常定义：

I/O输入/输出(Input/Output)，分为IO设备和IO接口两个部分。
在POSIX兼容的系统上，例如Linux系统，I/O操作可以有多种方式，比如DIO(Direct I/O)，AIO(Asynchronous I/O，异步I/O)，Memory-Mapped I/O(内存映射I/O)等，不同的I/O方式有不同的实现方式和性能，在不同的应用中可以按情况选择不同的I/O方式。

IO编程[廖雪峰的博客](https://www.liaoxuefeng.com/wiki/0014316089557264a6b348958f449949df42a6d3a2e542c000/001431917590955542f9ac5f5c1479faf787ff2b028ab47000)：

> 在计算机中指Input/Output，也就是输入和输出。由于程序和运行时数据是在内存中驻留，由CPU这个超快的计算核心来执行，涉及到数据交换的地方，通常是磁盘、网络等，就需要IO接口。比如你打开浏览器，访问新浪首页，浏览器这个程序就需要通过网络IO获取新浪的网页。浏览器首先会发送数据给新浪服务器，告诉它我想要首页的HTML，这个动作是往外发数据，叫Output，随后新浪服务器把网页发过来，这个动作是从外面接收数据，叫Input。所以，通常，程序完成IO操作会有Input和Output两个数据流。当然也有只用一个的情况，比如，从磁盘读取文件到内存，就只有Input操作，反过来，把数据写到磁盘文件里，就只是一个Output操作。
> IO编程中，Stream（流）是一个很重要的概念，可以把流想象成一个水管，数据就是水管里的水，但是只能单向流动。Input Stream就是数据从外面（磁盘、网络）流进内存，Output Stream就是数据从内存流到外面去。对于浏览网页来说，浏览器和新浪服务器之间至少需要建立两根水管，才可以既能发数据，又能收数据。
> 由于CPU和内存的速度远远高于外设的速度，所以，在IO编程中，就存在速度严重不匹配的问题。举个例子来说，比如要把100M的数据写入磁盘，CPU输出100M的数据只需要0.01秒，可是磁盘要接收这100M数据可能需要10秒，怎么办呢？有两种办法：
> 第一种是CPU等着，也就是程序暂停执行后续代码，等100M的数据在10秒后写入磁盘，再接着往下执行，这种模式称为同步IO；
> 另一种方法是CPU不等待，只是告诉磁盘，“您老慢慢写，不着急，我接着干别的事去了”，于是，后续代码可以立刻接着执行，这种模式称为异步IO。
> 同步和异步的区别就在于是否等待IO执行的结果。好比你去麦当劳点餐，你说“来个汉堡”，服务员告诉你，对不起，汉堡要现做，需要等5分钟，于是你站在收银台前面等了5分钟，拿到汉堡再去逛商场，这是同步IO。
> 你说“来个汉堡”，服务员告诉你，汉堡需要等5分钟，你可以先去逛商场，等做好了，我们再通知你，这样你可以立刻去干别的事情（逛商场），这是异步IO。
> 很明显，使用异步IO来编写程序性能会远远高于同步IO，但是异步IO的缺点是编程模型复杂。想想看，你得知道什么时候通知你“汉堡做好了”，而通知你的方法也各不相同。如果是服务员跑过来找到你，这是回调模式，如果服务员发短信通知你，你就得不停地检查手机，这是轮询模式。总之，异步IO的复杂度远远高于同步IO。
> 操作IO的能力都是由操作系统提供的，每一种编程语言都会把操作系统提供的低级C接口封装起来方便使用。

java io 体系如下图：

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/JAVA_IO体系.png)

## 流

定义：

    代表任何有能力产出数据的数据源对象（能read）或者是有能力接收数据的接收端对象(能write)。
    在java中，已经将流抽象成输入输出对象，就像水管一样，将两个容易连接起来。
    流是一组有顺序的，有起点和终点的字符集合，是对数据传输的总称或抽象。


本质：

    数据传输，根据数据传输特性将流抽象为各种类，方便更直观的进行数据操作。

作用：

    为数据源和目的地建立一个输送通道。
    
特性：

	相对于程序来说，输出流是往存储介质或数据通道写入数据，而输入流是从存储介质或数据通道中读取数据，一般来说关于流的特性有下面几点：

	1. 先进先出，最先写入输出流的数据最先被输入流读取到。

	2. 顺序存取，可以一个接一个地往流中写入一串字节，读出时也将按写入顺序读取一串字节，不能随机访问中间的数据。（RandomAccessFile可以从文件的任意位置进行存取（输入输出）操作）

	3. 只读或只写，每个流只能是输入流或输出流的一种，不能同时具备两个功能，输入流只能进行读操作，对输出流只能进行写操作。在一个数据传输通道中，如果既要写入数据，又要读取数据，则要分别提供两个流。 

 

## java流

进入正题之前，我们先看张图，以便有更深的了解。

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/JAVA流结构.png)

Java的IO模型设计非常优秀，它使用[Decorator(装饰者)模式](https://baike.baidu.com/item/%E8%A3%85%E9%A5%B0%E6%A8%A1%E5%BC%8F/10158540?fr=aladdin)，按功能划分Stream，您可以动态装配这些Stream，以便获得您需要的功能。

#### 分类

1.  按照数据类型分为：字节(byte)流和字符(char)流，数据流最小单元分别是字节/字符
2.  按照数据流向分为：输入(input)流和输出(output)流,采用数据流的目的就是使得输出输入独立于设备。
3.  按照数据来源，就比较多了

```
	File（文件）： FileInputStream, FileOutputStream, FileReader, FileWriter 
	byte[]：ByteArrayInputStream, ByteArrayOutputStream 
	Char[]: CharArrayReader,CharArrayWriter 
	String:StringBufferInputStream, StringReader, StringWriter 
	网络数据流：InputStream,OutputStream, Reader, Writer 
```

字符流的由来： Java中字符是采用Unicode标准，一个字符是16位，即一个字符使用两个字节来表示。为此，JAVA中引入了处理字符的流。因为数据编码的不同，而有了对字符进行高效操作的流对象。本质其实就是基于字节流读取时，去查了指定的码表。

#### 几大抽象类

##### InputStream

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/InputStream.png)

IO 中输入字节流的继承图可见上图，可以看出：

	InputStream是所有的输入字节流的父类，它是一个抽象类。
	ByteArrayInputStream、StringBufferInputStream(上图的StreamBufferInputStream)、FileInputStream是三种基本的介质流，它们分别从Byte数组、StringBuffer、和本地文件中读取数据。
	PipedInputStream是从与其它线程共用的管道中读取数据.
	ObjectInputStream和所有FilterInputStream的子类都是装饰流（装饰器模式的主角）。

InputStream中的三个基本的读方法

      abstract int read() ：读取一个字节数据，并返回读到的数据，如果返回-1，表示读到了输入流的末尾。
      intread(byte[]?b) ：将数据读入一个字节数组，同时返回实际读取的字节数。如果返回-1，表示读到了输入流的末尾。
      intread(byte[]?b, int?off, int?len) ：将数据读入一个字节数组，同时返回实际读取的字节数。如果返回-1，表示读到了输入流的末尾。off指定在数组b中存放数据的起始偏移位置；len指定读取的最大字节数。
      
其它方法

      long skip(long?n)：在输入流中跳过n个字节，并返回实际跳过的字节数。
      int available() ：返回在不发生阻塞的情况下，可读取的字节数。
      void close() ：关闭输入流，释放和这个流相关的系统资源。
      voidmark(int?readlimit) ：在输入流的当前位置放置一个标记，如果读取的字节数多于readlimit设置的值，则流忽略这个标记。
      void reset() ：返回到上一个标记。
      booleanmarkSupported() ：测试当前流是否支持mark和reset方法。如果支持，返回true，否则返回false。

流结束的判断：方法read()的返回值为-1时；readLine()的返回值为null时。

##### OutputStream

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/OutputStream.png)

IO 中输出字节流的继承图可见上图，可以看出：

    OutputStream是所有的输出字节流的父类，它是一个抽象类。
    ByteArrayOutputStream、FileOutputStream是两种基本的介质流，它们分别向Byte数组、和本地文件中写入数据。
	PipedOutputStream是向与其它线程共用的管道中写入数据。
    ObjectOutputStream和所有FilterOutputStream的子类都是装饰流。

outputStream中的三个基本的写方法

    abstract void write(int?b)：往输出流中写入一个字节。
    void write(byte[]?b) ：往输出流中写入数组b中的所有字节。
    void write(byte[]?b, int?off, int?len) ：往输出流中写入数组b中从偏移量off开始的len个字节的数据。

其它方法

    void flush() ：刷新输出流，强制缓冲区中的输出字节被写出。
    void close() ：关闭输出流，释放和这个流相关的系统资源。

##### Reader

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/Reader.png)

在上面的继承关系图中可以看出：

	Reader是所有的输入字符流的父类，它是一个抽象类。
	CharReader、StringReader是两种基本的介质流，它们分别将Char数组、String中读取数据。	PipedReader是从与其它线程共用的管道中读取数据。
	BufferedReader很明显就是一个装饰器，它和其子类负责装饰其它Reader对象。
	FilterReader是所有自定义具体装饰流的父类，其子类PushbackReader对Reader对象进行装饰，会增加一个行号。
	InputStreamReader是一个连接字节流和字符流的桥梁，它将字节流转变为字符流。FileReader可以说是一个达到此功能、常用的工具类，在其源代码中明显使用了将FileInputStream转变为Reader的方法。我们可以从这个类中得到一定的技巧。Reader中各个类的用途和使用方法基本和InputStream中的类使用一致。后面会有Reader与InputStream的对应关系。
	
主要方法：

     (1) public int read() throws IOException; //读取一个字符，返回值为读取的字符 
     (2) public int read(char cbuf[]) throws IOException; /*读取一系列字符到数组cbuf[]中，返回值为实际读取的字符的数量*/ 
     (3) public abstract int read(char cbuf[],int off,int len) throws IOException; 
    /*读取len个字符，从数组cbuf[]的下标off处开始存放，返回值为实际读取的字符数量，该方法必须由子类实现*/ 

##### Writer

![](https://raw.githubusercontent.com/lwl1989/javaTest/master/static/Writer.png)


在上面的关系图中可以看出：

	Writer是所有的输出字符流的父类，它是一个抽象类。
	CharArrayWriter、StringWriter是两种基本的介质流，它们分别向Char数组、String中写入数据。PipedWriter是向与其它线程共用的管道中写入数据，
	BufferedWriter是一个装饰器为Writer提供缓冲功能。
	PrintWriter和PrintStream极其类似，功能和使用也非常相似。
	OutputStreamWriter是OutputStream到Writer转换的桥梁，它的子类FileWriter其实就是一个实现此功能的具体类（具体可以研究一SourceCode）。功能和使用和OutputStream极其类似.
	
主要方法：

	public void write(int c) throws IOException； //将整型值c的低16位写入输出流 
	public void write(char cbuf[]) throws IOException； //将字符数组cbuf[]写入输出流 
	public abstract void write(char cbuf[],int off,int len) throws IOException； //将字符数组cbuf[]中的从索引为off的位置处开始的len个字符写入输出流 
	public void write(String str) throws IOException； //将字符串str中的字符写入输出流 
	public void write(String str,int off,int len) throws IOException； //将字符串str 中从索引off开始处的len个字符写入输出流 


#### 字符流和字节流的转换

转换流：

在IO中还存在一类是转换流，将字节流转换为字符流，同时可以将字符流转化为字节流。

	InputStreamReader:字节到字符的桥梁

	OutputStreamWriter:字符到字节的桥梁

 
	OutputStreamWriter:将字节流以字符流输出。

	InputStreamReader:将字节流以字符流输入。

