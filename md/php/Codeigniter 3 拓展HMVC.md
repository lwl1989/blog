CreateTime:2015-10-13 18:13:03.0

在Codeiniter（以下统称CI） 2.X版本中，我们就通过拓展核心类库实现了HMVC，但是同样的代码，拿到CI 3中，就很有可能不好用了。

###拓展核心类库方式

官方的文档对拓展核心类有详细的说明：
   
>你定义的类必须继承自父类。
你的类名和文件名必须以 MY_ 开头。（这是可配置的，见下文）
举个例子，要扩展原始的 Input 类，你需要新建一个文件 application/core/MY_Input.php，然后像下面这样定义你的类:

```
class MY_Input extends CI_Input {

}
```

CI 控制器加载的过程很简单，官方文档有图如下：

![输入图片说明](http://codeigniter.org.cn/user_guide/_images/appflowchart.png "在这里输入图片标题")
我们可以看到，在控制器开始加载看，CI是做了Routing（路由）和Security（安全）的操作的,所以，我们需要重写，或者说，在CI拓展我们想要的功能，比如：HMVC

###2.0中扩展

在2.0版本中，笔者曾适用过Jens Segers开源的HMVC模块，代码的实现就是对Routee和Loader进行了重写。
        [Jens Segers的主页](http://www.jenssegers.be/)
核心的代码如下：
```
            if (is_dir($source = $location . $module . '/controllers/')) {
                $this->module = $module;
                $this->directory = $relative . $module . '/controllers/';
                
                // 根目录下的模块？
                if ($directory && is_file($source . $directory . '.php')) {
                    $this->class = $directory;
                    return array_slice($segments, 1);
                }
                
                // 子模块？
                if ($directory && is_dir($source . $directory . '/')) {
                    $source = $source . $directory . '/';
                    $this->directory .= $directory . '/';
                    
                    // 子控制器？
                    if (is_file($source . $directory . '.php')) {
                        return array_slice($segments, 1);
                    }
                    
                    // 子文件夹包含有默认控制器？
                    if (is_file($source . $this->default_controller . '.php')) {
                        $segments[1] = $this->default_controller;
                        return array_slice($segments, 1);
                    }
                    
                    // 子文件夹中的控制器？ 
                    if ($controller && is_file($source . $controller . '.php')) {
                        return array_slice($segments, 2);
                    }
                }
                
                // 控制器和文件夹名一样？
                if (is_file($source . $module . '.php')) {
                    return $segments;
                }
                
                // 适用默认的控制器？
                if (is_file($source . $this->default_controller . '.php')) {
                    $segments[0] = $this->default_controller;
                    return $segments;
                }
            }
```
很简单的拓展
就能实现HMVC模式了，同样的，还得重写Loader中的加载器，不然会找不到文件。

###CI 3 HMVC拓展
到了CI3中，上述方法已经不好用了，CI 3 对路由有了更多的考虑，在初始化路由时，就进行了解析。假设写MY_Ruter类，必须要重写3个方法：
CI 2 Router构造
```
	function __construct()
	{
		$this->config =& load_class('Config', 'core');
		$this->uri =& load_class('URI', 'core');
		log_message('debug', "Router Class Initialized");
	}
```
CI 3 Router构造
```
public function __construct($routing = NULL)
	{
		$this->config =& load_class('Config', 'core');
		$this->uri =& load_class('URI', 'core');

		$this->enable_query_strings = ( ! is_cli() && $this->config->item('enable_query_strings') === TRUE);

		
		is_array($routing) && isset($routing['directory']) && $this->set_directory($routing['directory']);
		
		$this->_set_routing();

		
		if (is_array($routing))
		{
			empty($routing['controller']) OR $this->set_class($routing['controller']);
			empty($routing['function'])   OR $this->set_method($routing['function']);
		}

		log_message('info', 'Router Class Initialized');
	}
```

可以看到，在CI3的构造方法中就已经对URL进行解析，方法的调用过程为：
    _set_routing() -> _validate_request() ->   _parse_routes()    ->   _set_request()
那我们为了要实现HMVC，这几个方法是必然要按照我们自己的方法实现的。
_validate_request 中 我们加入部分验证，即可达到简单的HMVC
```
 if (is_dir($source = $relative . $module . '/controllers/')) {
                $this->module = $module;
                $this->directory =  '../'.$location.$module . '/controllers/';


                // 如果 有 application/$module/controollers/$directory.php 文件
                if ($directory && is_file($source . ucfirst($directory) . '.php')) {
                    return array_slice($segments, 1);
                }

                //如果application/$module/$directory 是一个文件夹
                if ($directory && is_dir($source . $directory . '/')) {
                    $source = $source . $directory . '/';
                    $this->directory .= $directory . '/';
                    //  index.php/$modules/$directory/$controller
                    //如果包含 控制器  $controller
                    if ($controller && is_file($source . ucfirst($controller) . '.php')) {
                        return array_slice($segments, 2);
                    }

                    //如果有默认控制器
                    if (is_file($source . $this->default_controller . '.php')) {
                        $segments[1] = $this->default_controller;
                        return array_slice($segments, 1);
                    }

                    //如果有 application/$module/$directory.php
                    if (is_file($source . $directory . '.php')) {
                        return array_slice($segments, 1);
                    }
                }

                //如果有 application/$module/$module.php  
                if (is_file($source . $module . '.php')) {
                    return $segments;
                }

                // 默认控制器
                if (is_file($source . $this->default_controller . '.php')) {
                    $segments[0] = $this->default_controller;
                    return $segments;
                }
            }
```

为了让CI_ROUTER知道我们模块拓展的位置，我们在配置文件中加入选项，并在CI_ROUTER的构造器中加入如下代码：
```
$locations = $this->config->item('modules_locations');

		if (!$locations) {
			$locations = array('modules/');
		} else if (!is_array($locations)) {
			$locations = array($locations);
		}

		$this->config->set_item('modules_locations', $locations);
```
操作完以上步骤，就可以实现大部分HMVC的拓展了。
    本文代码：https://git.oschina.net/liwenlong/Codeigniter-3-HMVC.git
说明：CI3中对控制器大小写由严格的控制了，为了符合CI3的一贯规则，所以我们使用了ucfirst()方式寻找首字母大写的类名，一定要注意。