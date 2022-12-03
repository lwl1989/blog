CreateTime:2016-03-31 20:11:26.0

##入门

    1.下载PHP源码（我的版本是5.6.13）
    2.常规编译安装  [可以看我以前的blog](http://my.oschina.net/lwl1989/blog/511295)
    3.使用ext_skel
    ```
        cd phpsrc/ext
        ./ext_skel --extname=name  // name 是指你要写得扩展的名字，下文我们都是t2
    ```
出现提示
```
1.  $ cd ..
2.  $ vi ext/t2/config.m4
3.  $ ./buildconf
4.  $ ./configure --[with|enable]-t2
5.  $ make
6.  $ ./sapi/cli/php -f ext/t2/t2.php
7.  $ vi ext/t2/t2.c
8.  $ make

```

根据提示进行操作
其中第二步
    我们应该修改config.m4的10、11、12行
    去除掉dnl[我的理解是ZEND一种注释的方案]

到这个时候，我们就可以进行编译安装了
```
/usr/local/php/bin/phpize   //执行phpize
./configure --with-php-config=/usr/local/php/bin/php-config
make 
make install
```
在php.ini里开启 t2.so文件  你的模块就加载好了
查看php -m 可以发现t2模块已经被加载





##编码（HELLO  WORLD）
但是目前而言，这个扩展是没有任何功能的。
于是我们需要加入新的功能。

这时，我们可以对t2.c文件进行编写（当然你也可以写在别的文件里，在t2.c里面include）

```
PHP_FUNCTION(t2_hello)
{
    printf("hello world\n");
}
```

这样重新编译后，运行 php -r "t2_hello();"是不行的，为什么呢？

因为我们没有将t2_hello载入到函数列表

原型：
```
typedef struct _zend_function_entry {
    char *fname;
    void (*handler)(INTERNAL_FUNCTION_PARAMETERS);
    unsigned char *func_arg_types;
} zend_function_entry;

```
> 参数	                描述
fname	              提供给PHP中调用的函数名。例如mysql_connect
handler	            负责处理这个接口函数的指针。
func_arg_types	     标记参数。可以设置为NULL



我们在生成的中加入一行

```
static const zend_function_entry t2_functions[] = {
    //在这加入
     {NULL, NULL, NULL}
}
```
    PHP_FE(t2_hello,  NULL)

> 注意，zend_function_entry的最后一项一定是{NULL, NULL, NULL}，Zend引擎就是靠这个判断导出函数列表是否完毕的。

然后重新编译，运行php -r "t2_hello();"
这时，我们的hello world!到此完成了


时间不太多，下次我们再说关于变量和参数值传递的处理还有几个模块的魔术方法(zend_module_entry)。