CreateTime:2015-11-02 16:19:09.0

_废话不多说了，直接看源代码把_

CI首先加载的是system/core/Codeigniter.php
阅读发现(165行)：
```
if ($composer_autoload = config_item('composer_autoload'))
	{
		if ($composer_autoload === TRUE)
		{
			file_exists(APPPATH.'vendor/autoload.php')
				? require_once(APPPATH.'vendor/autoload.php')
				: log_message('error', '$config[\'composer_autoload\'] is set to TRUE but '.APPPATH.'vendor/autoload.php was not found.');
		}
		elseif (file_exists($composer_autoload))
		{
			require_once($composer_autoload);
		}
		else
		{
			log_message('error', 'Could not find the specified $config[\'composer_autoload\'] path: '.$composer_autoload);
		}
	}
```
果然CI3根目录下的composer.josn不是拿来开玩笑的。
在配置文件里，我们同样可以看到是否开启composer。
```
/*
|--------------------------------------------------------------------------
| Composer auto-loading
|--------------------------------------------------------------------------
|
| Enabling this setting will tell CodeIgniter to look for a Composer
| package auto-loader script in application/vendor/autoload.php.
|
|	$config['composer_autoload'] = TRUE;
|
| Or if you have your vendor/ directory located somewhere else, you
| can opt to set a specific path as well:
|
|	$config['composer_autoload'] = '/path/to/vendor/autoload.php';
|
| For more information about Composer, please visit http://getcomposer.org/
|
| Note: This will NOT disable or override the CodeIgniter-specific
|	autoloading (application/config/autoload.php)
*/
```

那么很明显，CI3已经完全可以支持COMPOSER了。
为试验里一下，在composer.json里加入里阿里支付的接口
然后
    
    composer update;

在相应的控制器里加入代码
```
use mytharcher\sdk\alipay;
  function test1(){
        $this->pay_config = $this->config->item('alipay');
        $alipay = new \mytharcher\sdk\alipay\Alipay($this->pay_config);
        var_dump($alipay);
    }
```
果然打印出ali支付类的信息了。
但是发生里什么情况呢？
为了让我的视图和逻辑分开，我特别写了一个MY_View_Controller用于无逻辑的页面的加载
测试的时候，报错了。
```
Message: Class 'MY_View_Controller' not found

Filename: controllers/Welcome.php
```
找不到类？
经调试，关闭掉composer，就不会发生这个错误了。
为了解决这个错误，那么再次读源码把
既然是composer与autoload冲突，那么为只管看composer的代码就好了

    代码加载顺序  vendor/autoload.php  =>  vendor/composer/autoload_real.php  => vendor/composer/Classloader.php

终于，在Classloader类中（298行），发现了loadclass($class)方法  
好像和 __autoload($class)方法很像?
```
   public function loadClass($class)
    {
        if ($file = $this->findFile($class)) {
            includeFile($file);

            return true;
        }
    } 
```
那么我们在方法内部加上
```
  /**
         * 增加核心库依赖加载
         */
        if(strpos($class, 'MY_') === 0)
        {
            if (file_exists(APPPATH . 'core/'. $class . EXT)) {
                @include_once( APPPATH . 'core/'. $class . EXT );
            }

        }
```
果然好用了。


但是冲突的原因依旧没找到，肠炎犯了，下次再研究。
当然，为了让项目的其他人使用composer同样可以兼容这个错误，我们只需要再.gitignore文件中加入即可
```
vendor/
!vendor/autoload.php
!vendor/composer/*
```