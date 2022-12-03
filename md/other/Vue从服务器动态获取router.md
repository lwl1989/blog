CreateTime:2018-04-02 11:40:22.0

## 预先准备工作

- 提供接口
- 重写route

#### 提供接口

laravel提供接口那不是分分钟的是嘛
api.php
```
Route::group(['prefix'=>'admin','middleware'=>'format'],  function (){
        Route::get('routes','Manager\RouterController@getRouter');
});
```
然后编写一个类RouterController的getRouter方法
```
        public function getRouter()
        {
                $router = [
                        [
                                'path'=>'/',
                                'component'=>'Home',
                                'name'=>'臺東金幣',
                                'hidden'=>false,
                                'children'=>[
                                        [
                                                'path'           =>      '/admin/index',
                                                'component'      =>      'Admin',
                                                'name'           =>      '金幣賬戶'
                                        ],
                                        [
                                                'path'           =>      '/admin/index1',
                                                'component'      =>      'Admin',
                                                'name'           =>      '金幣發放'
                                        ],
                                        [
                                                'path'           =>      '/admin/index2',
                                                'component'      =>      'Admin',
                                                'name'           =>      '商品兌換'
                                        ]
                                ]
                        ]
                ];
                return ['router' => $router];
    }
```
很可靠，现在我们的接口返回的就是一段json了
```
{
    "code": 0,
    "response": {
        "router": [
            {
                "path": "/",
                "component": "Home",
                "name": "臺東金幣",
                "hidden": false,
                "children": [
                    {
                        "path": "/admin/index",
                        "component": "Admin",
                        "name": "金幣賬戶"
                    },
                    {
                        "path": "/admin/index1",
                        "component": "Admin",
                        "name": "金幣發放"
                    },
                    {
                        "path": "/admin/index2",
                        "component": "Admin",
                        "name": "商品兌換"
                    }
                ]
            }
        ]
    }
}
```
至于为什么我的数据格式自动变成了这样，请参照，[laravel(5.5)自定义middle ware](https://my.oschina.net/lwl1989/blog/1786390)

## 百度不可靠

[segmentfault](https://segmentfault.com/a/1190000009396901)

[csdn](https://blog.csdn.net/s8460049/article/details/61190709)

这是百度来的一系列文章，但是，我应用之后根本就报错，so，只好自己去看文档了。

案例中的 route.addRoutes(); 不可用


## 被文档坑的骚年

#### route push坑坑篇

[route push](https://router.vuejs.org/zh-cn/essentials/navigation.html)

按照往常的想法，push肯定是添加一系列路由，但是，不好意思，这里真不是。

> router.push(location, onComplete?, onAbort?)

> 注意：在 Vue 实例内部，你可以通过 $router 访问路由实例。因此你可以调用 this.$router.push。

> 想要导航到不同的 URL，则使用 router.push 方法。

> 这个方法会向 history 栈添加一个新的记录，所以，当用户点击浏览器后退按钮时，则回到之前的 URL。

我使用了之后，居然反复请求了接口。


#### 导航完成前获取数据

咦，这不就是我们想要的？
```
 beforeRouteUpdate (to, from, next) {
    this.post = null
    getPost(to.params.id, (err, post) => {
      this.setData(err, post)
      next()
    })
  },
```
完，看完文档之后，根本不是想要的东西，这里是加入了一个临时状态，比如loading。

## 解决方案
最后，终于找到了解决方案，官方把它叫[导航守卫](https://router.vuejs.org/zh-cn/advanced/navigation-guards.html#%E7%BB%84%E4%BB%B6%E5%86%85%E7%9A%84%E5%AE%88%E5%8D%AB)???

#### beforeEach

```
const myRouter = new Router({routes});
myRouter.beforeEach((to, from, next) => {
    if(loading == false) {   //只加载一次
        axios.get('/api/admin/routes').then(function (response) {
            if (response.data.code == 0) {
                let rou = response.data.response.router;
                rou.forEach((item, index) => {
                    console.log(rou);
                    myRouter.options.routes.push(item);
                });
                loading = true;
                next();
            }else{
                window.location.href = '/';
            }
        }).catch(function (error) {
            window.location.href = '/';
        });
    }
});
```

总算是完美的解决了这个问题。
下一步，就是解决动态获取路由失败的方案了。下次见。

## 总结


1. 文档大哥，你能说的清楚点吗？
1. 百度google不可尽信，必须先demo。
1. 前端技术真是日新月异啊，完全跟不上了（作为一个只是偶尔兼职一下前端的大后端）

## 新增

发现这种方式加载route会使route的点击事件失效

修改方式，登录后，讲route写入浏览器缓存。比如storge

然后第一次加载的时候，从storge获取。


```
let router = sessionStorage.getItem("router");

if(router == null || router == '') {
    window.location.href = '/login';
}
try {
    router = JSON.parse(router);
}catch (e) {
    window.location.href = '/login';
}

routes.forEach((r,k)=>{
    if ("children" in r && r.children.length > 0) {
        r.children.forEach((son,key)=>{
            if (router.indexOf(son.name)) {
                    r.hidden = false;
                son.hidden = false;
            }

        });
    }
});
```