CreateTime:2017-08-10 22:11:43.0

> 高层模块不应该依赖于底层模块，两个都应该依赖抽象。

>抽象不应该依赖于细节，细节应该依赖于抽象。

### 首先，我们来看一段代码：

```
class A{
        public function echo()
        {
                echo 'A'.PHP_EOL;
        }
}
class EchoT {
        protected  $t;
        public function __construct()
        {
              $this->t = new A();
        }
        public function echo(){
                $this->t->echo();
        }
}
``` 
初始，我们都使用new 的方式在内部进行，EchoT类严重依赖于类A。每当类A变化时，EchoT类也得进行变化。

### 我们优化一下代码

```
class EchoT {
        protected  $t;
        public function __construct($t)  //构造器注入由构造器注入到其中
        {
              $this->t = $t;
        }
```
可以看到，这样做的话。很大程序上，我们对程序进行了解耦。类A无论你如何变动，EchoT类是不需要变动的。不再依赖于A。但是新问题又来了，我们现在只有A,万一来了B，来了CDEFG怎么办。

### 面向接口

```
interface T{
        public function echo();
}

class A{
        public function echo()
        {
                echo 'A'.PHP_EOL;
        }
}

class B implements T{
        public function echo()
        {
                echo 'B'.PHP_EOL;
        }
}
class EchoT {
        protected  $t;
        public function __construct(T $t)  //构造器注入由构造器注入到其中
        {
              $this->t = $t;
        }
        public function echo(){
                $this->t->echo();
        }
}
```
将T抽象出为接口，这样，EchoT类中的echo方法变成一个抽象的方法，不到运行那一刻，不知道他们的Method方式是怎么实现的。

### 工厂

```
function getT($str) {
    if(class_exists($str)){
        return new $str();
        }
}
```
T要使用哪个是不明确的，因此，我们可以将其工厂化。【看上去很简单，在DI实际上有体现】

### DI（重点来了）

首先，我们看一下PHP的psr规范。

>http://www.php-fig.org/psr/psr-11/

官方定义的接口
```
Psr\Container\ContainerInterface
包含两个方法
function get($id);
function has($id);
```
仔细看上面的工厂，是不是和get($id)很一致，PHP官方将其定义为容器(Container,我个人理解，就是一个复杂的工厂)

#### dependency injection container
依赖注入容器
```
namespace Core;
use Psr\Container\ContainerInterface;
class Container implements ContainerInterface
{
        protected $instance = [];//对象存储的数组
        public function __construct($path) {
                $this->_autoload($path);  //首先我们要自动加载  psr-autoload
        }

        public function build($className)
        {
                if(is_string($className) and $this->has($className)) {
                        return $this->get($className);
                }
                //反射
                $reflector = new \ReflectionClass($className);
                if (!$reflector->isInstantiable()) {
                        throw new \Exception("Can't instantiate ".$className);
                }
                // 检查类是否可实例化, 排除抽象类abstract和对象接口interface
                if (!$reflector->isInstantiable()) {
                        throw new \Exception("Can't instantiate ".$className);
                }
                /** @var \ReflectionMethod $constructor 获取类的构造函数 */
                $constructor = $reflector->getConstructor();
                // 若无构造函数，直接实例化并返回
                if (is_null($constructor)) {
                        return new $className;
                }
                // 取构造函数参数,通过 ReflectionParameter 数组返回参数列表
                $parameters = $constructor->getParameters();
                // 递归解析构造函数的参数
                $dependencies = $this->getDependencies($parameters);
                // 创建一个类的新实例，给出的参数将传递到类的构造函数。
                $class =  $reflector->newInstanceArgs($dependencies);
                $this->instance[$className] = $class;
                return $class;
        }

        /**
         * @param array $parameters
         * @return array
         */
        public function getDependencies(array $parameters)
        {
                $dependencies = [];
                /** @var \ReflectionParameter $parameter */
                foreach ($parameters as $parameter) {
                        /** @var \ReflectionClass $dependency */
                        $dependency = $parameter->getClass();
                        if (is_null($dependency)) {
                                // 是变量,有默认值则设置默认值
                                $dependencies[] = $this->resolveNonClass($parameter);
                        } else {
                                // 是一个类，递归解析
                                $dependencies[] = $this->build($dependency->name);
                        }
                }
                return $dependencies;
        }

        /**
         * @param \ReflectionParameter $parameter
         * @return mixed
         * @throws \Exception
         */
        public function resolveNonClass(\ReflectionParameter $parameter)
        {
                // 有默认值则返回默认值
                if ($parameter->isDefaultValueAvailable()) {
                        return $parameter->getDefaultValue();
                }
                throw new \Exception($parameter->getName().' must be not null');
        }
        /**
         * 参照psr-autoload规范
         * @param $path
         */
        public function _autoload($path) {
                spl_autoload_register(function(string $class) use ($path) {
                        $file = DIRECTORY_SEPARATOR.str_replace('\\',DIRECTORY_SEPARATOR, $class).'.php';
                        if(is_file($path.$file)) {
                                include($path.$file);
                                return true;
                        }
                        return false;
                });
        }

        public function get($id)
        {
                if($this->has($id)) {
                        return $this->instance[$id];
                }
                if(class_exists($id)){
                        return $this->build($id);
                }
                throw new ClassNotFoundException('class not found');  //实现的PSR规范的异常
        }

        public function has($id)
        {
                return isset($this->instance[$id]) ? true : false;
        }
}
```

#### 使用示例

```
$container = new Container('../');//假设这是路径
$echoT = $container->get(\Test\EchoT::class);     //假设echoT类的命名空间是\Test
$echoT->echo();
```
这个时候，会出现一个问题：
```
  // 检查类是否可实例化, 排除抽象类abstract和对象接口interface
                if (!$reflector->isInstantiable()) {
                        throw new \Exception("Can't instantiate ".$className);
                }
因为接口T是无法实例化的，所以，一般在程序内，我们都加上别名（参照laravel框架）
$container->alisa(\Test\T::class,\Test\T\A::class);  //指定接口T使用类A(控制反转)
```
#### 针对接口

下面是alias方法
```
      public function alias(string $key, $class, bool $singleton = true) 
        {
                if($singleton) {
                        $this->singleton[] = $class;
                }
                $this->aliases[$key] = $class;
                return $this;
        }
    //同时，我们需要在build的时候进行判断是否为别名
 public function build($className)
        {
                if(is_string($className) and $this->has($className)) {
                        return $this->get($className);
                }
                if(isset($this->aliases[$className])) {
                        if(is_object($this->aliases[$className])) {
                               return $this->aliases[$className];
                        }
                        $className = $this->aliases[$className];
                }
```

就此，一个简单的PHP容器就实现了。

### 个人实现代码

[我最近一个爬虫项目（基于swoole）](https://github.com/lwl1989/agileSwoole)


### 参考：

[PHP之道](http://wulijun.github.io/php-the-right-way/)

[编程老头](http://www.cnblogs.com/sweng/p/6392336.html)

[PHP程序员如何理解依赖注入容器(dependency injection container)](https://segmentfault.com/a/1190000002424023)