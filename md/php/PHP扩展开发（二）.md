CreateTime:2016-04-01 19:42:50.0

> 上文我们学会了如何快速的进行一个PHP扩展的hello world!
下面我们将学习如何传递参数

###必要知识点

1.变量存储结构(php 5.6  src/ZEND/zend.h)
```
typedef union _zvalue_value {
        long lval;       // long value
        double dval;     // double value
        struct {
                char *val;
                int len;
        } str;
        HashTable *ht;    //
        zend_object_value obj;
        zend_ast *ast;
} zvalue_value;

struct _zval_struct {
        // Variable information 
        zvalue_value value;             // value 
        zend_uint refcount\__gc;
        zend_uchar type;                // active type 
        zend_uchar is_ref\__gc;
};
```

最终形成一个[这个就是PHP的变量]
typedef struct _zval_struct zval;
其实际值存储在  zval.value
判断是否为引用  is_ref__gc
判断引用的次数  refcount  gc  [PHP的写时复制特性]
判断值的类型    type


###源码分析
我们打开zend_API.c
可以看到很多关于变量的处理的函数。

比如 
ZEND_API int zend_get_parameters(int ht, int param_count, ...) 

ZEND_API int _zend_get_parameters_array(int ht, int param_count, zval **argument_array TSRMLS_DC）

ZEND_API int zend_get_parameters_ex(int param_count, ...)

ZEND_API int _zend_get_parameters_array_ex(int param_count, zval ***argument_array TSRMLS_DC)

ZEND_API int zend_parse_parameters ......


我们来参考一下原来的代码。
打开文件 phpsrc/ext/standard/var.c

我们看PHP是如何实现var_dump方法的
```
  87 PHPAPI void php_var_dump(zval **struc, int level TSRMLS_DC) /* {{{ */
  88 {
  89     HashTable *myht;
  90     const char *class_name;
  91     zend_uint class_name_len;
  92     int (*php_element_dump_func)(zval** TSRMLS_DC, int, va_list, zend_hash_key*);
  93     int is_temp;
  94 
  95     if (level > 1) {
  96         php_printf("%*c", level - 1, ' ');
  97     }
  98 
  99     switch (Z_TYPE_PP(struc)) {
 100     case IS_BOOL:
 101         php_printf("%sbool(%s)\n", COMMON, Z_LVAL_PP(struc) ? "true" : "false");
 102         break;

           ......
    }


173 line 
PHP_FUNCTION(var_dump)
 173 {
 174     zval ***args;
 175     int argc;
 176     int i;
 177 
 178     if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "+", &args, &argc) == FAILURE) {
 179         return;
 180     }
 181 
 182     for (i = 0; i < argc; i++) {
 183         php_var_dump(args[i], 1 TSRMLS_CC);
 184     }
 185     efree(args);
 186 }

```

可以发现 和上节说的一样
首先定义 PHP_FUNCTION(函数名){实现}
那我们发现没有参数怎么办？

这里就要用到我们开始说的方法了  178行
**zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "+", &args, &argc)**

明显由其获得了
参数  和  参数的个数

然后调用了函数的实现体

到此，PHP扩展开发怎么调用函数，思路就已经很明确了。


###实践
我们还是依照昨天的example,今天我们写一个A+B的方法

首先，我们定义一个方法

```
    PHP_FUNCTION(example_add){
        long a,b;
        if (zend_parse_parameters(ZEND_NUM_ARGS() TSRMLS_CC, "ll", &a, &b) == FAILURE) {
              return;
        }
        
        //添加要做的操作
       
    }
```

我们加入我们要实现的A+B功能


printf("%d\n",a+b);


同样，我们添加到functions
```
static const zend_function_entry t2_functions[] = {
    //在这加入方法的定义
    PHP_FE(example_add,NULL)
     {NULL, NULL, NULL}
}
```

重新编译安装，我们就可以运行
```
<?php example_add(3,5); ?>  
```
输出8