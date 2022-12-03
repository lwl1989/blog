CreateTime:2016-03-17 11:41:12.0

###About  thrift
----
The Apache Thrift software framework, for scalable cross-language services development, combines a software stack with a code generation engine to build services that work efficiently and seamlessly between C++, Java, Python, PHP, Ruby, Erlang, Perl, Haskell, C#, Cocoa, JavaScript, Node.js, Smalltalk, OCaml and Delphi and other languages.

总体而言，就是开源的、跨平台的、无缝连接的开源框架。一般用于系统内个语言之间的RPC通信。

[源代码位置](https://git-wip-us.apache.org/repos/asf/thrift)

###为什么产生了thrift
----

    目前主流的数据传输格式：
    xml与JSON相比体积太大，但是xml传统，也不算复杂。
    json 体积较小，新颖，但不够完善。
    thrift 体积超小，使用起来比较麻烦，不如前两者轻便，但是对于
        1.高并发
        2.数据传输量大
        3.多语言环境
    满足其中2点使用 thrift还是值得的。


    
    和其他协议对比（引用自https://www.liuhe36.cn/2013/07/introduction-of-thrift/）：
    其实有很多技术能支撑远程调用，如常见的REST-JSON方式、REST-XML或RMI等，但REST方式的效率上确实不够高，下面引用了几张图片，可以直观的说明一些问题。

    使用Thrift和其他方式的所产生的内容大小比较结果如下：
![![[thrift-size](http://https://www.liuhe36.cn/wp-content/uploads/2013/07/thrift-size.png)](http://https://www.liuhe36.cn/wp-content/uploads/2013/07/thrift-size.png "在这里输入图片标题")](https://www.liuhe36.cn/wp-content/uploads/2013/07/thrift-size.png "在这里输入图片标题")

    在上图中我们能明显看出，最臃肿的是RMI，其次是xml，使用Thrift的TCompactProtocol协议和Google 的 Protocol Buffers 相差的不算太多，相比而言还是Google 的 Protocol Buffers效果最佳。

    使用Thrift 中的协议和其他方式的所产生的运行开销比较结果如下：
![[thrift-load](https://www.liuhe36.cn/wp-content/uploads/2013/07/thrift-load.png)](https://www.liuhe36.cn/wp-content/uploads/2013/07/thrift-load.png "在这里输入图片标题")

    在上图中我们能明显看出，最占资源是REST2中协议，使用Thrift的TCompactProtocol协议和Google 的 Protocol Buffers 相差的不算太多，相比而言Thrift的TCompactProtocol协议效果最佳。

###数据类型

####基本类型：

    bool：布尔值，true 或 false，对应 Java 的 boolean       PHP bool
    byte：8 位有符号整数，对应 Java 的 byte                 PHP  int
    i16：16 位有符号整数，对应 Java 的 short                PHP  int
    i32：32 位有符号整数，对应 Java 的 int                  PHP  int
    i64：64 位有符号整数，对应 Java 的 long                 PHP  int
    double：64 位浮点数，对应 Java 的 double                PHP  double
    string：未知编码文本或二进制字符串，对应 Java 的 String  PHP  string

####结构体类型：

    struct：定义公共的对象，类似于 C 语言中的结构体定义，在 Java 中是一个 JavaBean,在PHP里是一个类

####容器类型：

    list：对应 Java 的 ArrayList   PHP  array
    set：对应 Java 的 HashSet      PHP  array
    map：对应 Java 的 HashMap      PHP  array

####异常类型：

    exception：对应 Java 的 Exception PHP 的 Exception

####服务类型：

    service：对应服务的类