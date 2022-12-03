CreateTime:2018-03-28 17:22:34.0

###  middleware [什么是middleware](http://laravelacademy.org/post/2803.html)

>HTTP 中间件提供了为过滤进入应用的 HTTP 请求提供了一套便利的机制。例如，Laravel 内置了一个中间件来验证用户是否经过授权，如果用户没有经过授权，中间件会将用户重定向到登录页面，否则如果用户经过授权，中间件就会允许请求继续往前进入下一步操作。

>当然，除了认证之外，中间件还可以被用来处理更多其它任务。比如：CORS 中间件可以用于为离开站点的响应添加合适的头（跨域）；日志中间件可以记录所有进入站点的请求。

>Laravel框架自带了一些中间件，包括维护模式、认证、CSRF 保护中间件等等。所有的中间件都位于 app/Http/Middleware 目录。


### 如何定义中间件
    
    php artisan make:middleware middleName

或者按照目录规范在 app/Http/Middleware下建立新文件 middleName.php

### 代码示例

```
namespace App\Http\Middleware;

use Closure;

class OldMiddleware
{
    /**
     * 返回请求过滤器
     *
     * @param \Illuminate\Http\Request $request
     * @param \Closure $next
     * @return mixed
     */
    public function handle($request, Closure $next)
    {
        if ($request->input('age') <= 200) {
            return redirect('home');
        }

        return $next($request);
    }
}
```
很明显，代码就是说，当你提交的年龄小于20，就跳转到home。

### 前置和后置
```
 public function handle($request, Closure $next)  {
       //todo 这里增加逻辑是前置
        $res =  $next($request);
        //todo 这里增加逻辑是后置
    }
```

### 我的一个json format的应用
```
public function handle($request, Closure $next)
    {
            $response = $next($request);


            if($response instanceof  JsonResponse) {
                    $data = $response->getData(true);

                    if($response->getStatusCode() != 200) {
                            if(!isset($data['code'])) {
                                    $response->setData(['code'=>1, 'response'=>$data]);
                                    return $response;
                            }
                    }
                    if(!isset($data['code'])) {
                            $response->setData(['code'=>0, 'response'=>$data]);

                    }
                    return $response;
            }

            if(is_array($response)) {
                    if(!isset($response['code'])) {
                           return ['code'=>0,'response'=>$response];
                    }
            }

            return $response;
    }
```

可以看到，我这里是一个对controller的结果做的一个后置的middleware。具体的作用是：

如果一个代码的返回值是一个JsonResponse对象，那么，判断返回值里有没有code下标的值，如果没有，将返回值重写。如果有，直接返回（还判断了http code）。

### 联合Route::group的应用
       
结合上面的middleware，我们可以将路由的返回值全部格式化。
比如：

        Route::group( ['prefix'=>'admin','middleware'=>'format'],  function (){

        Route::get('count','Manager\AdminController[@count](https://my.oschina.net/Count)');

        //......
        
        });
比如：
```
class AdminController extends Controller
{
         public function count()
        {
                return ['count'=>100];
        }
}
```

明显的，我们返回的只是{"count"=>"100"}

那我们打开测试网址返回的却是：
```
{
    "code": 0,
    "response": {
        "count": 3
     }
}
```
很完美的做到了我们想要的结果。假如数据错误，我们要返回一个我们已经的错误号呢？

我们改写一下代码
```
 public function count()
        {
               $a= 0;
               if($a==0) {
                       return ['code'=>'10000','count'=>0];
               }
               return ['count'=>count(self::USER)];
        }
```
返回值如下：
```
{
    "code": "10000",
    "count": 0
}
```
更多的功能，比如，错误号对应的提示，我们也可以通过middleware将其封装进去，这样可以大大降低开发的成本。