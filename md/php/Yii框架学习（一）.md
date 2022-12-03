CreateTime:2016-03-17 16:05:42.0

近期，由于项目需求，接触到了yii框架，便想着把日常学习到的东西写出来给大家分享。

###入门

####Yii 版本

Yii 当前有两个主要版本：1.1 和 2.0。
接下来的系列均为2.0。

####系统要求和先决条件

Yii 2.0 需要 PHP 5.4.0 或以上版本支持。你可以通过运行任何 Yii 发行包中附带的系统要求检查器查看每个具体特性所需的 PHP 配置。

####安装
    
    必要条件 安装composer、php5.4+

```
composer global require "fxp/composer-asset-plugin:~1.0.0"
composer create-project --prefer-dist yiisoft/yii2-app-basic basic
```

这里会要求生成一个token,请到github中生成.

官方提示：
    注意：在安装过程中 Composer 可能会询问你 GitHub 账户的登录信息，因为可能在使用中超过了 GitHub API （对匿名用户的）使用限制。因为 Composer 需要为所有扩展包从 GitHub 中获取大量信息，所以超限非常正常。（译注：也意味着作为程序猿没有 GitHub 账号，就真不能愉快地玩耍了）登陆 GitHub 之后可以得到更高的 API 限额，这样 Composer 才能正常运行。更多细节请参考 Composer 文档（该段 Composer 中文文档期待您的参与）。

官方的提示应该是已经不符合现在的GITHUB安全机制了，现在是生成唯一token。

####hello word!
    安装完毕之后，修改 config/web.php 文件，给 cookieValidationKey值。
    
    打开：http://localhost/basic/web/index.php,就有如下效果

![输入图片说明](http://www.yiichina.com/docs/guide/2.0/images/start-app-installed.png "在这里输入图片标题")

这里的代码位于 controllers/SiteController.php

```
<?php
    namespace app\controllers;
    use yii\web\Controller;

    class SiteController extends Controller
    {
    // ...其它代码...
        public function actionSay($message = 'Hello')
        {
            return $this->render('say', ['message' => $message]);
        }
    }
```

我们将return $this->render('say', ['message' => $message]);  修改为
    echo 'hello,world!';

刷新，赤果果的入门程序就来了！