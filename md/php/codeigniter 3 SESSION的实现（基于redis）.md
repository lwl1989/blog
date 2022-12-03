CreateTime:2015-10-12 16:23:32.0

###CI 2 SESSION的诟病
相信无数人在使用CI2的Session类库时，遇到各种的坑，各种抱怨，各种不解。在CI中国论坛能搜到大量关于Session类库的提问，说明要想用好session类库还是得下一番功夫。
####Session和cookie的区别
在某些语境中，cookie是session的一种实现方式，Ci 2的类库设计似乎就这么认为的。于是，产生了CI2中COOKIE即SESSION的说法。在安全性方面，CI 2当然也由考虑，COOKIE是经过加密过的，而且一旦修改，服务器便不再识别。

**CI 2的session工作原理
**

CI 2提供了一个元数据，我们可以把这个元数据看为一个自定义的验证机制，其包含以下内容：
```
Array
(
    [session_id] => 4a5a5dca22728fb0a84364eeb405b601
    [ip_address] => 127.0.0.1
    [user_agent] => Mozilla/5.0 (Macintosh; U; Intel Mac OS X 10_6_7;
    [last_activity] => 1303142623
)
```
其中Session id是重要的，Session id不对，直接拒绝。
验证选项在配置文件里有规定，（IP地址的限制）（user agent限制）（上次活动时间）等
```
$config['sess_match_ip']  = FALSE;
$config['sess_match_useragent'] = TRUE;
$config['sess_time_to_update'] = 300;
```
特别是当设置sess_match_useragent设为TRUE时，会遇到各种的坑：
你用flash组件上传文件，只有登录的用户才能上传文件，结果你每次判断用户是否登录都会出错，因为flash发送的http请求有可能更改了user agent；
使用ie切换不同的模式，比如兼容模式，也会造成user agent不同；
user agent的长度最长是120个字符，手动设置User agent是需要截取字符到最大120个。
另外，如果出现“Cannot modify header information - headers already sent by”错误，基本可以断定是你文件的编码格式有误，请去掉bom头。

####安全问题

CI使用cookie来传递数据本来就不够安全，而且如果数据量巨大时也会有性能问题。但是CI还是友好地加入以下几个安全验证机制：

**加密令牌**


    $config['encryption_key'] = 'mahuaz_';
    $config['sess_encrypt_cookie'] =  TRUE;

设置后，客户端检查cookie时就可以看到加密后的序列化数据。

**自动更新机制**

CI默认每五分钟更新一次令牌，更新发生在客户端的一次请求中。客户端每发送一次请求，会把cookie的信息发送到服务器，服务器根据发来的cookie判断是否到了应该更换令牌的时间了，如果是，就会重新换一个新的令牌返回给客户端。这就相当于门卫给你换了令牌，下次要使用新令牌进门。此时即使有坏人伪造了一个令牌也不起作用了，因为旧令牌已经作废。这样就相当于加了一条安全机制。可以使用：


    $config['sess_time_to_update'] = 300;

来设置多长时间来换一次令牌。这个时间不要设置的太短，更新频繁也会影响性能。这里不要和sess_expiration混淆：


    $config['sess_expiration']  = 7200;

这个配置是用来指示整个cookie的过期时间的，相当于令牌完全失效，再怎么更换都不起作用。

###CI 3的变更
CI3的Session的重大改变就是默认使用了原生的Session，这符合Session类库本来的意思，似乎更加合理一些。总体来说，虽然设计理念不同，但为了保证向后兼容性，类库的使用方法与CI2.0的差别不是很大。一般的使用过程是这样的：

截取一段CI2 SESSION的代码：
```
         if ($this->sess_encrypt_cookie == TRUE)
                {
                        $this->CI->load->library('encrypt');
                }

                if ($this->sess_use_database === TRUE AND $this->sess_table_name != '')
                {
                        $this->CI->load->database();
                }


                $this->now = $this->_get_time();

  
                if ($this->sess_expiration == 0)
                {
                        $this->sess_expiration = (60*60*24*365*2);
                }


                $this->sess_cookie_name = $this->cookie_prefix.$this->sess_cookie_name;


```

截取一段CI3 SESSION的代码：
```
	$class = $this->_ci_load_classes($this->_driver);

		// Configuration ...
		$this->_configure($params);

		$class = new $class($this->_config);
		if ($class instanceof SessionHandlerInterface)
		{
			if (is_php('5.4'))
			{
				session_set_save_handler($class, TRUE);
			}
			else
			{
				session_set_save_handler(
					array($class, 'open'),
					array($class, 'close'),
					array($class, 'read'),
					array($class, 'write'),
					array($class, 'destroy'),
					array($class, 'gc')
				);

				register_shutdown_function('session_write_close');
			}
		}
		else
		{
			log_message('error', "Session: Driver '".$this->_driver."' doesn't implement SessionHandlerInterface. Aborting.");
			return;
		}

```
可以看到CI 3已经完全重写了SESSION，由不同的驱动器用来保存SESSION（并且淘汰了COOKIE）。

###CI3 SESSION FOR REDIS
在配置文件中，CI 3的配置也完全缩减了很多
```
$config['sess_driver'] = 'files';
$config['sess_cookie_name'] = 'ci_session';
$config['sess_expiration'] = 7200;
$config['sess_save_path'] = NULL;
$config['sess_match_ip'] = FALSE;
$config['sess_time_to_update'] = 300;
$config['sess_regenerate_destroy'] = FALSE;
``` 
CI 3 可以配置驱动器类型，包括files, database, redis, memcached以及自定义，$config['sess_driver'] 来配置驱动器，默认的驱动器是files。官方推荐用files类型（在一般情况下）
配置session save path
配置节sess_save_path会根据不同的驱动器，定义不同。
为了适用redis,我这里将他配置为
    
    tcp://127.0.0.1:6379
在上面的代码中，有下面这两句

```
$class = new $class($this->_config);
		if ($class instanceof SessionHandlerInterface){
...
}
```
确立了$_SEESION 由 redis进行存储，加载的类是Session_redis_driver(位于sytstem/libraries/Session_redis_driver),在测试控制器中，
我们写一些测试代码：

```
    $_SESSION['test'] = 'There is test!';
    var_dump($_SESSION);
```
会打印出

    array(2) { ["__ci_last_regenerate"]=> int(1444637502) ["test"]=> string(13) "There is test" } 
我们在Session_redis_driver的read方法中写到
    
    var_dump($this->_key_prefix.$session_id);  //redis  key的值，由_key_prefix.$session_id组合构成
会打印出
    
    string(51) "ci_session:23ea4298dc5e7d6b808ca70ddb1665590e5cfb58"

为了测试redis,我们可以在PHP中测试：

```
$redis = new redis();
$redis->connect('127.0.0.1',6379);
var_dump($redis->get("ci_session:23ea4298dc5e7d6b808ca70ddb1665590e5cfb58"));
```
会打印出

    string(60) "__ci_last_regenerate|i:1444637502;test|s:13:"There is test";"

同样，我们在redis-cli中输入

    get ci_session:23ea4298dc5e7d6b808ca70ddb1665590e5cfb58
会打印出

    __ci_last_regenerate|i:1444637502;test|s:13:"There is test
至此，CI 3 使用REDIS作为SESSION存储就分析完毕了