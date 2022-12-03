CreateTime:2018-04-09 18:37:27.0

> 本文Laravel版本为5.5

# laravel中的实现方式

laravel自带了完整的auth方案，可以参照文章[laravel的应用](http://laravelacademy.org/post/8270.html)。我这里实现举出几个重要的步骤:

```
    //生成代码
    php artisan make:auth
    //导入数据表结构
    php artisan migrate
```

重要的文件：

    //用户模型
    app/Http/User.php
    
    //业务逻辑
    app/Http/Controllers/Auth/*.php
    
    //配置文件
    config/auth.php

我们观察一下配置文件，个人的理解我加上注释写在后面：
```
<?php
return [
    'defaults' => [                      /*默认配置*/ 
        'guard' => 'web',                /*guard（我个人看成检验器，翻译是守卫、监视者） 使用web的配置 */
        'passwords' => 'users',           /*密码管理，使用users的配置*/
    ],
    'guards' => [                                  
        'web' => [                     /*web的检验器的配置，有驱动器和提供者2个属性*/
            'driver' => 'session',
            'provider' => 'users',
        ],
        'api' => [                      /*api的检验器的配置，有驱动器和提供者2个属性*/
            'driver' => 'token',
            'provider' => 'users',
        ],
    ],
    'providers' => [                     /*提供者的驱动器和模型*/
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Http\User::class,
        ]
    ],
    'passwords' => [
        'users' => [                    /*密码管理的提供者属性*/
            'provider' => 'users',
            'table' => 'password_resets',
            'expire' => 60,
        ],
    ]
];
```
从配置来看，我们可以把laravel的auth看成多个部分(而passwords应该属于其中一项功能):
1. 入口(web|api|......)
2. 验证器guards
3. 提供者providers

# 分析源码
以登录为例，我们开始进入源码阅读，当我们点进去看LoginController.php时，发现他没有几行代码。但是他用了一个trait，至于什么是trait，请参照[php的trait](http://php.net/manual/zh/language.oop5.traits.php)。里面也只是些简单的业务逻辑实现方法，如果不需要，可以重写，或者注释。

但是，我从构造函数发现了，他使用了middleware，如果你不懂什么叫middleware，请参照我个人的blog[laravel(5.5)自定义middle ware](https://my.oschina.net/lwl1989/blog/1786390)。

    $this->middleware('guest')->except('logout');
很明显，他使用了中间件进行处理，我们进行代码追踪，就可以进入下一步了。

#### laravel是如何注入guard
    
guest这个中间件，定义在app/Http/Kernel.php中。

    'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class

主要的代码如下：
```
 public function handle($request, Closure $next, $guard = null)
    {
        if (Auth::guard($guard)->check()) {
            return redirect('/');
        }

        return $next($request);
    }
```
那，我们找到了我们要找的Auth。Auth是一个门面，也就是一个快捷使用的类的入口，我们找到他的实现类的位置\Illuminate\Auth\AuthServiceProvider，终于算是找到我们的正主了。
```
    protected function registerAuthenticator()
    {
        $this->app->singleton('auth', function ($app) {
            // Once the authentication service has actually been requested by the developer
            // we will set a variable in the application indicating such. This helps us
            // know that we need to set any queued cookies in the after event later.
            $app['auth.loaded'] = true;

            return new AuthManager($app);
        });

        $this->app->singleton('auth.driver', function ($app) {
            return $app['auth']->guard();
        });
    }
```
每当请求进入的时候，只要使用了Auth这个门面的地方，都会注册这个验证身份。实现类，就是AuthManager类。在这个类中，有这么一个方法，将验证器注入。
```
    public function guard($name = null)
    {
        $name = $name ?: $this->getDefaultDriver();
        return $this->guards[$name] ?? $this->guards[$name] = $this->resolve($name);
    }
```
发现了嘛？driver就是从这里写入的，配合配置文件中的driver就会生成一个driver类。不过这个代码质量也不是很高，比如下面的：
    
      $driverMethod = 'create'.ucfirst($config['driver']).'Driver';

emmm，硬生生的字符串拼接获取方法名？？？好把，这里按照laravel这么“优雅的框架”，不应该加上工厂模式嘛？额，小小吐槽一次。

#### laravel的验证器是如何工作的
接下来，我们就可以看他的验证器是如何实现的了。以TokenGuard为例把。
```
   public function createTokenDriver($name, $config)
    {

        $guard = new TokenGuard(
            $this->createUserProvider($config['provider'] ?? null),
            $this->app['request']
        );

        $this->app->refresh('request', $guard, 'setRequest');

        return $guard;
    }
```
返回了一个TokeGuard的类，细看下去，里面无非是判断有没有值提交[api_key|api_token]，没有的话就去找http的验证头Authorization Bearer或者$_SERVER['PHP_AUTH_PW']，然后，重要的地方来了。
```
    public function validate(array $credentials = [])
    {
        if (empty($credentials[$this->inputKey])) {
            return false;
        }
        $credentials = [$this->storageKey => $credentials[$this->inputKey]];
        if ($this->provider->retrieveByCredentials($credentials)) {
            return true;
        }
        return false;
    }

``` 
这里直接查表？条件就是key= input key的模式，实在太恐怖了，查表至少也要用主键把？

# 修改重写
从上面的分析来看，大多数的功能代码已经分析完毕了，我们开始着手重写这个复杂的轮子把。重写，当然是先实现底层类比较好。

### model的重写
当一个用户的属性来源于多张表的时候，一个模型肯定是不够了，但是laravel的Auth，显然是不方便拓展的。因此，我们只能改写一部分。
我们找到laravel实现的地方！回想开始的配置，默认使用的驱动器是eloquent，类是App\Http\User。
找到实现类Illuminate\Auth\uentUserProvider。里面实现了多种查询方式，那我们发现了一个规律，查询之前，他一定是先创建模型，模型就是我们的类App\Http\User。
比如：
```
 $query = $this->createModel()->newQuery();
        foreach ($credentials as $key => $value) {
            if (! Str::contains($key, 'password')) {
                $query->where($key, $value);
            }
        }

return $query->first();
```
找了很久，我发现，我们不好改写这个类，因为这个类并没有用配置去注入，但是，我找到了一条规律，查询之前，必然会先newQuery()，这个方法的调用对象是我们配置的。
于是，我的改写版出路，我改写了App\Http\User的newQuery()
```
    public function  newQuery() {
                $query = $this->newBaseQueryBuilder();
                $builder =  new class($query) extends Builder {

                        /**
                         * Create a new Eloquent query builder instance.
                         *
                         * @param  \Illuminate\Database\Query\Builder  $query
                         * @return void
                         */
                        public function __construct(QueryBuilder $query)
                        {

                                parent::__construct($query);
                        }
                        /**
                         * Execute the query and get the first result.
                         *
                         * @param  array  $columns
                         * @return \Illuminate\Database\Eloquent\Model|object|static|null
                         */
                        public function first($columns = ['*']) {
                                /** @var Model $user */
                                $user =  $this->take(1)->get($columns)->first();
                                //從別的模型查數據  設置到這個模型裏面
                                if($user!=null) {
                                        //拓展user的其他屬性
                                        $user->setAttribute('acl', ['dsadas', 'dsadas', 'dsada']);
                                }
                                //var_dump($user);exit();
                                return $user;
                        }

                };
                $builder->setModel($this);
                return $builder;
        }
```
是不是，用户信息存在于多表的问题就这么简单的解决了。

### guard的重写

但是，比如上面的接口，一般调用会很勤快，这个时候，我们还每次用这么乱的资料去查库，是不是会导致db卡IO（并发的时候）。一般情况，我们已经知道了用户的ID，接口一般都会附带用户信息进行请求，不是么？那么，我们很显然，只需要根据ID进行查询用户信息即可。

首先，我们新建类App\Library\Auth\TokenGuard以备接下来使用。

再看之前的TokeGuard类，我们发现他的条件是使用的token进行查询的，我们不妨把他改成主键，但是主键从何而来？所以我们要先获取到user_id。我设想，我的token是通过user_id加密而来，这时候，我只需要解密就好了。
```
    $this->uid = $this->_validateToken($token);  //伪代码
    public function validate(array $credentials = [])
    {
        if($this->provider->retrieveById($this->uid)) {  //直接改成从ID查询
            return true;
        }
        return false;
    }
```
显然，既然我们已经知道ID，用ID查询岂不美滋滋，如果对性能要求特别高，而且不想在数据库进行查询，完全可以把这个部分实现在redis，这里不是我们的重点，我就不细讲了（实现还是修改newQuery修改成redis查询即可，记得返回Model对象即可）。

### AuthManager的重写
既然验证器也写好了，我们肯定要把自己的验证器覆盖。
新建一个类,重写createTokenDriver方法用自己的类覆盖即可
```
class AuthManager extends \Illuminate\Auth\AuthManager
{
        public function createTokenDriver($name, $config)
        {
                $guard = new TokenGuard(
                        $this->createUserProvider($config['provider'] ?? null),
                        $this->app['request']
                );

                $this->app->refresh('request', $guard, 'setRequest');

                return $guard;
        }
}
```
###  其他配置
最后，我们需要把我们的认证服务加载到内核中来。
重写App\Providers\AuthServiceProvider集成原本的认证类，只需重写registerAuthenticator,其它的功能就照样可以用。
```
class AuthServiceProvider extends \Illuminate\Auth\AuthServiceProvider
{
        protected function registerAuthenticator()
        {
                $this->app->singleton('auth', function ($app) {
                        $app['auth.loaded'] = true;

                        return new \App\Library\Auth\AuthManager($app);
                });

                $this->app->singleton('auth.driver', function ($app) {
                        return $app['auth']->guard();
                });
        }
}

```
config/app.php
修改
```
Illuminate\Auth\AuthServiceProvider::class
=》
App\Providers\AuthServiceProvider::class
```
# 总结

### 个人心得
其实个人不太喜欢这种方式，毕竟登录注册这种含有业务逻辑的东西，我觉得封装到框架内部的意义并不大。但是为了偷懒的话，改写复用也是一种方式。

### 备注
注意，修改密码和更改更新之类的，需要在update之前unset那么其它表的数据，才能进行更新，否则会报字段不存在。

### 源码位置
一个正在实现的教材管理系统，替俺一个关系很好的老师做的[edu_book](https://github.com/lwl1989/edu_book)。
还有公司的一个项目也共用这种方法重写auth，就不能开源了