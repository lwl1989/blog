CreateTime:2018-03-28 15:23:36.0

## 问题出现背景
    
公司接到一个项目，恩，外包。本来我是不想做，但是因为这个项目是政府单位，所以无奈得做了。然后，项目想要快速成型，于是我选择使用目前最流行的PHP框架laravel（之前公司都是用自己的框架，很精简）。

ok,始于laravel。

## 始于laravel

打开Laravel的文档，按照官方的quick start开始了一个项目，但是从根目录发现了package.json。

emmm,这不是nodejs的生态链吗？

于是百度google开始了自己的爬坑之旅。

## laravel+vue+element-ui

#### 环境

首先，我们需要构建一个新的vue过程，我简单的来(前提环境php composer node npm)
    
    composer install
    npm install
    
为了加快速度，一般我们会采用cnpm
    
    npm install -g cnpm --registry=https://registry.npm.taobao.org

好，想想，我们要用哪几个组件
    
    vue
    vue-router
    vue-resource //我并没有采用 axios 习惯问题
    element-ui
全部安装
    cnpm install 上面的包

执行laravel本地服务
    
    php artisan serve

#### php

添加路由(我这里使用了一个middleware，如果不用请忽略。)

```
Route::group(['prefix'=>'admin','middleware'=>'format'], function (){
        Route::get('count','Manager\AdminController@count');
        Route::get('index','Manager\AdminController@index');
        Route::get('select','Manager\AdminController@select');
        Route::post('create','Manager\AdminController@create');
        Route::put('update','Manager\AdminController@update');
        Route::delete('delete','Manager\AdminController@delete');
});
```
新建类，代码我就省略了

新建视图
```
   
    <div id="app">
        <admin-component>
        </admin-component>
    </div>
    
```

#### vue
新建vue组件
```
<template>
    <div id="app">
        <el-row>
            <el-col :span="24">
             这里是admin
            </el-col>
        </el-row>
    </div>
</template>

<script>
    export default {
        name: "adminIndexComponent",
}
</script>
```

RouteCompent.vue router的界面
```
<template>
    <div id="app">
        <router-view></router-view>
    </div>
</template>
```

新建入口admin.js
```
    import ELE from 'element-ui' 
    import VueResource from 'vue-resource'
    import App from  './components/RouteComponent.vue'
    import VUE from 'vue'
    window.Vue = VUE;
    const app = new Vue({
        router,
        render: h => h(App)
    }).$mount('#app');
```
新建router/index.js
```
    import Vue from 'vue'
    import Router from 'vue-router'
    import Admin from '../components/admin/AdminIndexComponent.vue'
    import Home from '../components/CommonComponent.vue'
    const routes = [
    {
        path: '/',
        component: Home,
        hidden: true
    },
    {
        path: '/',
        component: Home,
        name: '管理员列表',
        iconCls: 'el-icon-d-caret',
        hidden: false,
        children: [
            { path: '/admin/index', component: Admin, name: '管理员列表' },
            { path: '/admin/index', component: Admin, name: '管理员列表1' }
        ]
    }
    ];
    const router = new Router({ routes });

    export default router
```

关于laravel的crsf问题

    Vue.http.headers.common['X-CSRF-TOKEN'] = document.querySelector('meta[name=csrf-token]').getAttribute('content');

#### 打包js
首先我们要修改mix的打包结构,根目录下有webpack.mix.js,如下:

    mix.js('resources/assets/js/app.js', 'public/js')
         .sass('resources/assets/sass/app.scss', 'public/css')

我们依葫芦画瓢修改成自己想要的

    .js('resources/assets/js/admin.js', 'public/js')
    .sass('resources/assets/sass/admin.scss', 'public/css/admin.css')
        
    npm run dev

## laraver-mix优化
 打包完成后，发现包有1.3m。这肯定是不行的，政府的服务器是自己的老旧机器，我们都懂的。那该如何优化呢？
#### mix.extarct
我找到官方文档，说，可以将固定的功能单独打包。

     .extract(['vue','vue-router','vue-resource','element-ui'])
这样打包之后，会产生3个js
    
    js/manifest.js
    js/vendor.js
    js/admin.js
用mix引入即可
但是又发现vendor.js太大了。这个时候我去看了下源码，发现extract可以指定生成的名字，那我觉得应该也是可以分开打包。
于是：

    .extract(['element-ui'],'public/js/element-ui.js')
    .extract(['vue','vue-router','vue-resource'],'public/js/vue.js')
 
然而,我发现光element-ui打包出来就有700k，这样不行啊。继续寻找其他方式。
官方说**laravel-mix是基于webpack的上层封装**，然后我看了element-ui[按需引入](http://element-cn.eleme.io/#/zh-CN/component/quickstart),再次修改代码，只引入我要用的东西

admin.js
```
import { Submenu,Menu,MenuItemGroup,MenuItem,DropdownMenu,Dropdown,DropdownItem, Button , Input, Select, Dialog, Pagination, Table, TableColumn} from 'element-ui';

Vue.use(VueResource);
Vue.use(Button);
Vue.use(Input);
Vue.use(Select);
Vue.use(Dialog);
Vue.use(Pagination);
Vue.use(Table);
Vue.use(TableColumn);
Vue.use(Dropdown);
Vue.use(DropdownItem);
Vue.use(DropdownMenu);
Vue.use(Menu);
Vue.use(MenuItemGroup);
Vue.use(MenuItem);
Vue.use(Submenu);
```
这时候，再打包，我发现element-ui还是将整个包完全打好。emmm，为什么为什么。

## webpack config注入
再回到上面说的(加粗黑体字"laravel-mix是基于webpack的上层封装")，那么对webpack的修改肯定也会影响到laravel-mix
所以，继续百度google（element官网说的用blade我尝试过，laravel-mix没找到使用他的方式，可能是我太菜把）。

最后在mix找到一个方法，mix.webpackConfig,再找了一个关于element-ui的webpack打包的说明
```
mix.webpackConfig({
    module: {
        rules: [
            {
                test: /\.js$/,
                exclude: /node_modules/,
                loader: 'babel-loader',
                options: {
                    "plugins": [["component", [
                        {
                            "libraryName": "element-ui",
                            "styleLibraryName": "theme-chalk"  //原文theme-default 修改成theme-chalk  原文对css的使用css-loader取消
                        }
                    ]]]
                }
            },
      ]
    },
});
```
再次 npm install prod,duangduangduang!!!!
```
fonts/vendor/_element-ui@2.2.2@element-ui/lib/theme-chalk/element-icons.woff?2fad952a20fbbcfd1bf2ebb210dccf7a    6.16 kB          [emitted]         
 fonts/vendor/_element-ui@2.2.2@element-ui/lib/theme-chalk/element-icons.ttf?6f0a76321d30f3c8120915e57f7bd77e      11 kB          [emitted]         
                                                                                                 /js/admin.js     338 kB       0  [emitted]  [big]  /js/admin
                                                                                                   /js/vue.js     131 kB       1  [emitted]         /js/vue
                                                                                              /js/manifest.js  798 bytes       2  [emitted]         /js/manifest
                                                                                               /css/admin.css     193 kB       0  [emitted]         /js/admin

```

## nginx最后的优化
最后还有一个300kb+的文件，这时候，再采取我们的nginx进行优化（gzip）,[原文](https://blog.csdn.net/hjh15827475896/article/details/53432823)

```
                gzip on;
                gzip_buffers 32 4k;
                gzip_comp_level 6;
                gzip_min_length 200;
                gzip_types text/css text/xml application/javascript;

```

最后执行结果:

    原来338kb的文件，现在已经缩成了70kb,完美解决~


## 后记
   关于node-sass报错解决方案：[需要重新编译](https://www.cnblogs.com/chenliyang/p/6552625.html)

```
    cd PROJECT_PATH/node_modules/node-sass/vendor/linux-x64-59(自动产生的版本目录)
    当前目录会产生一个binding.node  
    但是好像用npm只会产生一个binding  我猜测直接改名也可以
    blog里说的是用 rebuild node-sass重新编译  也是可以的。

```