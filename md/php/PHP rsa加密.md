CreateTime:2016-11-30 18:25:00.0

##为什么要进行数据加密
嘿嘿，这个问题就和裸奔、裸睡一样，裸奔不好，裸睡可以。

裸奔谁都能看到你的小秘密，丁丁太小会被发现噢。而裸睡不同，裸睡你是在家，门和窗是API但是都有相应的保护措施。裸睡在家更自由，速度更快更敏捷，但是在外面裸奔你可要注意了，万一有仇视裸奔行为的人发现了你酱紫，那就对不起啦，割掉你的小丁丁。

所以，我们还是不要裸奔啦，让我们穿上衣服吧！

##常用加密方式
我个人把加密看做4大类
1. 对称加密
2. 非对称加密
3. 散列加密
4. 其他

对称加密常用的一般为：des  aes ...

非对称机密： rsa...

散列加密： md5 hash...

##RSA加密
RSA的来由可以参考[阮一峰的网络日志 ](http://http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)

我就不详细介绍了，说一说RSA在PHP的实现吧

##PHP实现

###获取公匙和私匙
首先，我们得有衣服，没有衣服怎么穿（公匙和私匙），对吧！获取方式：

1. linux shell

        genrsa -out rsa_private_key.pem 1024
        rsa -in rsa_private_key.pem -pubout -out rsa_public_key.pem
会产生2个文件，rsa_private_key.pem，rsa_public_key.pem

2. 用PHP产生

        $config = [
                 "private_key_bits" => 1024,
                 "private_key_type" => OPENSSL_KEYTYPE_RSA,
        ];
        $r = openssl_pkey_new($config);
        openssl_pkey_export($r, $privateKey);  //产生私钥
        $rp = openssl_pkey_get_details($r);
        $pubKey = $rp['key'];     //得到公钥

### RSA的加解密过程（使用公钥加密、私钥解密）

1. composer lwl1989/rsa 示例

             $rsa = new RSAGenerate();
             $publicKey = $rsa->getPublicKey();
             $privateKey = $rsa->getPrivateKey();
             $serverProvider = new Provider(['public_key'=>$publicKey,'private_key'=>$privateKey]);
             $pass =  $serverProvider->publicKeyEncode('12345');
              $pass = $serverProvider->decodePublicEncode($pass);

2. 分析加密过程

		$public_key = openssl_pkey_get_public($this->_config['public_key']);  //设置公匙
		openssl_public_encrypt($data, $encrypted, $public_key);                    //使用openssl进行加密
		return !empty($encrypted)?base64_encode($encrypted):'';                 //对结果再次base64

   其实base64是不需要的，只是为了保证数据的正确性而为之，可以看出，最重要的2步为：

        设置公匙
        以公匙进行加密

3. 分析解密过程

 同样还会有2次base64噢

    decodePublicEncode

		$private_key = openssl_pkey_get_private($this->_config['private_key']);     //设置公匙
		openssl_private_decrypt(base64_decode($data), $decrypted, $private_key); //私钥解密
		return !empty($decrypted)?$decrypted:'';
    可以看出
        同样还是2步
        设置私匙
        以私匙解密

##其他
本文代码位于 [GITHUB](https://github.com/lwl1989/RSA)

可以直接使用composer require "lwl1989/rsa"进行安装

JS推荐使用jsencrypt，github上可以搜到,在我的github上我也会完善一个demo!