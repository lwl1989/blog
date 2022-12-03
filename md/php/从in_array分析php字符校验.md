CreateTime:2019-10-14 20:57:11.0

# PHP in_array的坑

ps: 应该是弱类型语言的坑

[php文档](http://www.php.net/manual/pt_BR/function.in-array.php)

顾名思义，in_array就是查找一个值是否在数组里面。

## 问题

### 假设事故现场

一个sql注入的测试代码如下:

```
$type = $_GET['type'];
$types = [2,3,4,5,6];
if(!in_array($type, $types)) {
	throw new ParamsException('参数错误');
}

$sql = sprintf('select * from test where `type` = %s', $type);

$result = $db->fetch($sql);
```

emm，让我们理一下。

1. 从参数获取type
2. 检验type合法性
3. 拼sql
4. 查找数据

好像是这么回事。

但是，实际情况，这个接口将有sql注入风险。


### 问题分析查找

为什么会存在sql注入的风险呢？

ok，我们先了解下什么是常见的sql注入。

> 所谓SQL注入，就是通过把SQL命令插入到Web表单提交或输入域名或页面请求的查询字符串，最终达到欺骗服务器执行恶意的SQL命令。具体来说，它是利用现有应用程序，将（恶意的）SQL命令注入到后台数据库引擎执行的能力，它可以通过在Web表单中输入（恶意）SQL语句得到一个存在安全漏洞的网站上的数据库，而不是按照设计者意图去执行SQL语句。 [1]  比如先前的很多影视网站泄露VIP会员密码大多就是通过WEB表单递交查询字符暴出的，这类表单特别容易受到SQL注入式攻击．


举个栗子：

select * from a where id = 10;  正常sql

select * from a where id = 10 or 1;  注入sql

换到我们的场景而言，就是 type = 3 变成了 type = 3 or 1,那岂不是把所有值都返回了。

嗯？？？但是我们不是校验了type吗？怎么会被这样注入呢。

好，实践是检验真理的唯一标准。

```
$arr = [3,4,5];
$str = '3 or 1';
var_dump(in_array($str, $arr));
```

打印出来bool(true)!!!!

想必都惊呆了，原来是in_array校验数据出了问题。

再进行我的猜测。

```
$str = '3 or 1';
$int = intval($str);
echo $int; //3
var_dump($a == $int); // bool(true)
```

soga，原来，检测是这么容易通过的。我推断是因为在进行值比价的时候，因为没有使用强类型验证，所以，会将 3 or 1 == 3，从而判定是是在数组里。

然后继续仔细查看文档，发现in_array有第三个参数，作用是强类型判断是否相等（严格校验）。但是我想，还是不适合我的场景，我的场景是允许字符串传入的。

### 解决问题

既然问题找到了，解决就自然有简单了。

解决方案还是不说了，因为不同场景各位看官要用的解决方式也不一样。

思路基本是一样的，就是保证不会被异常数据注入。

## 源码关注

[github](https://github.com/php/php-src/blob/master/ext/standard/array.c)

```
PHP_FUNCTION(in_array)
{
	php_search_array(INTERNAL_FUNCTION_PARAM_PASSTHRU, 0);
}
```

ps: array_search最终调用的也是php_search_array

```
	zval *value,				/* value to check for */
		 *array,				/* array to check in */
		 *entry;				/* pointer to array entry */
	zend_ulong num_idx;
	zend_string *str_idx;
	zend_bool strict = 0;		/* strict comparison or not */

	ZEND_PARSE_PARAMETERS_START(2, 3)
		Z_PARAM_ZVAL(value)
		Z_PARAM_ARRAY(array)
		Z_PARAM_OPTIONAL
		Z_PARAM_BOOL(strict)
	ZEND_PARSE_PARAMETERS_END();

	if (strict) {
		//强类型验证......
		//这里用的fast_is_identical_function(value, entry))
	} else {
	   //可以观察到，php是先检测数组的值得类型，而不是根据查找值得类型进行比较
		if (Z_TYPE_P(value) == IS_LONG) {
		//long验证，在php里是整形
			if (fast_equal_check_long(value, entry)) {
		} else if (Z_TYPE_P(value) == IS_STRING) {
			
		}//...其他类型
```

而 fast_equal_check_string的原型如下

```
static zend_always_inline int fast_equal_check_long(zval *op1, zval *op2)
{
	zval result;
	if (EXPECTED(Z_TYPE_P(op2) == IS_LONG)) {
		return Z_LVAL_P(op1) == Z_LVAL_P(op2);
	}
	compare_function(&result, op1, op2);
	return Z_LVAL(result) == 0;
}
```

可以看到，当2个参数类型不一样的时候，会走compare_function

compare_function是zend Api内核提供的，源码暂未追踪，但是根据官方的说明，结果和预测的一样。

具体请查看官方：[php对比2个值是否相等--比较运算符](http://www.php.net/manual/zh/language.operators.comparison.php)

## 后记

不要脱离了框架就没法写代码

不要只说新标准，通常都要维护很多老代码（用pdo不就能解决？）