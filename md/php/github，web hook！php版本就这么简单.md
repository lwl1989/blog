CreateTime:2020-05-11 21:03:09.0

# 前言

好吧，这个东西很多人都用烂了，但是我感觉不是自己总结写得总有点记不住，那么就自己写一下吧。

### 用在哪里？

懒人测试环境开发吧，想push的时候不用手动去触发。

比如我这

- git pull
- composer update
- apidoc xxxxxx
- chmod (测试机懒得搞用户，直接是root,这个需要给执行者权限，你看撒，一看就是php。)

懒人就像少动手，怎么办，找办法偷懒咯。

# 前置步骤准备工作

## github

1. 你需要一个github(gitlab也行，哪怕内网也行，只要你服务器和git是互通的)
2. 新建一个项目（或者是已有项目也行）
3. 进入项目的地址，找到设置（什么，没有设置？setting也找不到嘛？和我一样回家学学英文再出来装逼吧，嗯我刚学完）
4. webhook，我擦，藏这么深！

![](https://oscimg.oschina.net/oscnet/up-a20f4f1c7f78a2acc8430c2df6f7f8952f0.png)
紧接着你会发现出现这个玩意。

url懂吧，一看，他说要发送一个post请求给你的服务器，好让你的服务器自动执行XXX，可选的是application/json或者 x-www-form-urlencoded。

乖乖的就好，傻瓜模式，我还以为要干啥呢。

输入我们的地址： http://192.168.0.xx/webhook.php

?? 发现一个密匙？ 这是啥。

无奈只好点developer documentation.

噢 ，就酱紫

![](https://oscimg.oschina.net/oscnet/up-3e047224d5b79aa1a3754a622011938b9be.png)

然后，他问我们要触发啥？当然是更新就触发咯，push gogogo!

到此 github处理完毕。

## 服务器

1. 嗯，首先你得有台服务器。
2. ssh 啰嗦吧
3. git php composer nginx ... any more
4. 需要1个php接收脚本, [laravel版本演示代码](https://github.com/lwl1989/liuliu/blob/master/app/Http/Controllers/Controller.php)

```
<?php

// GitHub Webhook Secret.
// Keep it the same with the 'Secret' field on your Webhooks / Manage webhook page of your respostory.
$secret = "";

// Path to your respostory on your server.
// e.g. "/var/www/respostory"
$path = "";

// Headers deliveried from GitHub
$signature = $_SERVER['HTTP_X_HUB_SIGNATURE'];

if ($signature) {
  $hash = "sha1=".hash_hmac('sha1', file_get_contents("php://input"), $secret);
  if (strcmp($signature, $hash) == 0) {
    echo shell_exec("执行脚本");
    exit();
  }
}

http_response_code(404);

```

5. 需要一个执行任务的脚本，也就是我开始说的那些（注意权限咯）。


## 总结

![](https://oscimg.oschina.net/oscnet/up-db1aecaa68ff53cd75b71c23affbf4a27dc.png)

在经历了几次细微的调试之后，一切OK
gogogo.




