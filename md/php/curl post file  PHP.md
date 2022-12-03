CreateTime:2016-08-12 09:33:47.0

###网上示例方式
    使用@加上文件名
    如aa.png  => @aa.png  
###使用结果
    PHP5.5以下可行，PHP5.5以上进行了封装
###PHP5.5及以上版本
       php5.5及以上版本封装了一个CURLFile类，[详情](http://www.php.net/manual/en/class.curlfile.php)
###示例
```
$ch = curl_init('https://example.com');
$cfile = new CURLFile('aa.png','image/jpeg','image_name');   //建立文件
$post_data = array(
               "image_name" => "bar",
               "image" => $cfile
 );
	        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST,  0);//关闭认证
		curl_setopt($ch, CURLOPT_SSL_VERIFYPEER,  0);//关闭认证
              //curl_setopt($ch, CURLOPT_HTTPAUTH, CURLAUTH_BASIC);
               // curl_setopt($ch, CURLOPT_USERPWD, "");
                curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
                curl_setopt($ch, CURLOPT_POST, 1);
                curl_setopt($ch, CURLOPT_POSTFIELDS, $post_data);
                $response = curl_exec($ch);
                curl_close($ch);
                echo $response;exit;
```