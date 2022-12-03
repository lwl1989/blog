CreateTime:2018-03-12 17:07:21.0

## 引言
pecl、pear、composer真是傻傻让人分不清啊


## PCEL、PEAR和COMPOSER

百科是这样形容PCEL
> PECL 的全称是 The PHP Extension Community Library ，是一个开放的并通过 PEAR(PHP Extension and Application Repository，PHP 扩展和应用仓库)打包格式来打包安装的 PHP扩展库仓库。通过 PEAR 的 Package Manager 的安装管理方式，可以对 PECL 模块进行下载和安装。

 百科是这样形容PEAR(和composer作用类似，但是目前好像composer占据主流)
>全称为PHP扩展与应用库(PHP Extension and Application Repository)。为了创建一个类似于Perl CPAN档案的工具，Stig S. Bakken在1999年创立了PEAR项目。


百科是这样形容的composer
>Composer 是 PHP5.3以上 的一个依赖管理工具。它允许你声明项目所依赖的代码库，它会在你的项目中为你安装他们。Composer 不是一个包管理器。是的，它涉及 "packages" 和 "libraries"，但它在每个项目的基础上进行管理，在你项目的某个目录中（例如 vendor）进行安装。默认情况下它不会在全局安装任何东西。因此，这仅仅是一个依赖管理。


## 如何安装pear

centos:

    yum install php-pear

ubuntu:

    apt install php-pear

假如你的PHP是编译进行安装的，用系统自带的安装方式会出现新的问题，此时PHP不是你想要的版本（系统库通常会低于发行的新版本很多），而且会自动安装依赖，就是会新装一次PHP并且全局变量覆盖。

因此还提供了一个新的安装方式：

    curl -o go-pear.php http://pear.php.net/go-pear
    php go-pear.php

我（PHP7.2）执行下发现报错，说我的PHP版本太新，初始我没仔细看，太新我就用旧的，手动改phpversion为7.1,结果还是报错，这是没有仔细看报错的锅。

```
Sorry!  Your PHP version is too new (7.2.3) for this go-pear.
Instead use http://pear.php.net/go-pear.phar for a more stable and current
version of go-pear, more suited to your PHP version.

Thank you for your coopertion and sorry for the inconvenience!
```
仔细一看，原来是新版需要用新的脚本文件
    
    curl -o go-pear.php http://pear.php.net/go-pear
    php go-pear.php

这个时候pear就安装好了，我们需要给它创建一个软链接。

    ln -s /php_root/bin/pear /usr/bin/pear


## 如何安装composer

composer 安装就相对简单了很多：

    curl -sS https://getcomposer.org/installer | php
    mv composer.phar /usr/local/bin/composer

```

## PECL安装扩展
pcel install redis

.................

Build process completed successfully
Installing '/usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/redis.so'
install ok: channel://pecl.php.net/redis-3.1.6
configuration option "php_ini" is not set to php.ini location
You should add "extension=redis.so" to php.ini
```

可以看到，不需要其他操作，可以很快的装好一个扩展。大多数PHP扩展已经加入到pear,当然也有不存在的，这个时候还是得手动进行编译安装（可以查看我另一篇blog[PHP手动安装扩展](https://my.oschina.net/lwl1989/blog/534238)）。



## composer vs pear
composer和pear的功能非常像。都是用来对已有代码包的管理的。
在composer还没诞生之前，PHP的代码很难被管理。虽然pear社区的支持，许多可重用代码可以通过pear来获得，但是pear在处理代码关联性上非常差，当然还有许多问题，比如入库标准太高等。
composer以json文件的形式解决了包的依赖问题，而且加入了很多的规范，大大解决了pear未解决的问题，很大趋势上是完全替代了pear。

## 总结
所以总结下来：
    - pcel是安装php的C扩展程序的
    - pear是老一代包管理工具
    - composer是新一代包管理工具
    pear和composer都是对PHP代码进行管理，个人推荐composer。