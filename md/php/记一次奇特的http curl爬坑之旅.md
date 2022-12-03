CreateTime:2019-10-23 15:57:34.0

## 背景

写到一个项目，要做一个简单的持久化任务，于是乎，我用到了swoole（毕竟phper，而且排期紧，swoole开发速度还是比java、 go 快）。

## swoole 4

swoole 4 已经全量支持自动协程了，这点对于广大php开发者有很大的优势。

举个栗子吧。

```
$http->on('request',function (Swoole\Http\Request $req, Swoole\Http\Response $res) {
    //todo: 这里已经是协程调度了
});
```

用swoole的话就最好使用最新版本（当然可能文档没跟上，这点需要观众朋友自行解决，不建议php新手入手）。

4.4是长期支持版本，目标是打造成工业级的(现在时间：2019-10-23)

## 出现情况

我在项目里使用了 guzzlehttp/guzzle, 版本^6.3。

然后，从某天起，就出现了很奇特的bug。 

	Could not resolve: oapi.dingtalk.com (Successful completion)

代码如下：

```
        $client = new \GuzzleHttp\Client([
		'base_uri'  => 'https://oapi.dingtalk.com',
		'timeout'  => 5.0,
		'verify' => false,
        ]);

        try{
		$response = $client->request('POST', PATH, [
			 ......
		]);

		unset($client);
		$code = $response->getStatusCode();
		if ($code == 200) {
			$data = json_decode($response->getBody()->getContents(), true);
			return $data;
		}
		Logs::writeErrorLog('xxxx失败超时 code:' . $code, false);
		return [];
	}catch(Exception $e) {
		Logs::writeErrorLog('xxxx失败超时 error:' . $e->getMessage());
	}
```

catch总捕获到error？并且是dns解析不到？

## 问题排查

1. 浏览器 https://oapi.dingtalk.com 正常

2. curl https://oapi.dingtalk.com。

3. 同样参数，postman提交，正常。

4. 无奈重启，（原谅我，重启大法好）,重启后第一次正常，然后gg思密达

5. 把timeout去掉，结果报错是dns resolve失败？

6. 手写curl

神奇的时候来了,看代码。

```
        $ch = curl_init($url); // 启动一个CURL会话
        $useragent="Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1.1) Gecko/20061204 Firefox/2.0.0.1";

        curl_setopt($ch, CURLOPT_USERAGENT, $useragent);
        curl_setopt($ch, CURLOPT_CONNECTTIMEOUT, 4);
        curl_setopt($ch, CURLOPT_TIMEOUT, 8);
        curl_setopt($ch, CURLOPT_LOW_SPEED_TIME, 10);
        curl_setopt($ch, CURLOPT_HEADER, 0);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false); // 跳过证书检查
        curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, 2);  // 从证书中检查SSL加密算法是否存在
        curl_setopt($ch, CURLOPT_DNS_USE_GLOBAL_CACHE, false );
        curl_setopt($ch, CURLOPT_DNS_CACHE_TIMEOUT, 2 );

        $tmpInfo = curl_exec($ch);     //返回api的json对象
        $code = curl_errno($ch);
        $error = '';
        if($code != '') {
            $error = curl_error($ch);
        }
        $httpCode = curl_getinfo($ch,CURLINFO_HTTP_CODE);

        curl_close($ch);
        var_dump($tmpInfo, $code, $httpCode);
```

false , 6 ????

没错，code 6

	error : CURLE_COULDNT_RESOLVE_HOST (6)
	Couldn't resolve host. The given remote host was not resolved.

看来整个dns都整没了。

重试几次，一切都正常，唯独到resolve host这里出了问题。

## 解决方案

既然是解析不到host，那么，手动解析dns，拼接host主机就行了。

https://www.php.net/manual/zh/function.curl-setopt.php

观察到curl支持CURLOPT_RESOLVE。

CURLOPT_RESOLVE	提供自定义地址，指定了主机和端口。

包含主机、端口和 ip 地址的字符串，组成 array 的，每个元素以冒号分隔。

格式： array("example.com:80:127.0.0.1")	在 cURL 7.21.3 中添加，自 PHP 5.5.0 起可用。

那首先我们得得到主机的ip地址，上代码：

```
function getIp($host) : string{
        $ip = gethostbyname('oapi.dingtalk.com');
        if($ip == 'oapi.dingtalk.com') {
            return $ip;
        }

        $dns = dns_get_record('oapi.dingtalk.com');
        if(is_array($dns)) {
            foreach ($dns as $dn) {
                if(isset($dn['type']) and $dn['type'] == 'A') {
                    if(isset($dn['ip'])) {
                        return $dn['ip'];
                    }
                }
            }
        }
	return '';
}
```

然后，在上面的curl中加入参数。

```
$ip = getIp('oapi.dingtalk.com');
if($ip != '') {
 curl_setopt($ch, CURLOPT_RESOLVE, ['oapi.dingtalk.com:443:'.$ip]);
}
```

## 完成解决备注

1. 目前只在swoole协程中发现这个问题。
2. CURLOPT_RESOLVE可以充分应用在内网系统中（解决dns耗时）

## 附带curl其他坑

CURLOPT_TIMEOUT_MS: error


低版本CURL不支持CURLOPT_TIMEOUT_MS的bug;
```
if ( defined('CURLOPT_TIMEOUT_MS') && defined('CURLOPT_CONNECTTIMEOUT_MS') ) {
			$curl_opts[CURLOPT_TIMEOUT_MS] = max($timeout,1);
			$curl_opts[CURLOPT_CONNECTTIMEOUT_MS] = max($conn_timeout,1);
			$curl_opts[CURLOPT_NOSIGNAL] = true;
		}else {
			$curl_opts[CURLOPT_TIMEOUT] = max($timeout/1000,1);
			$curl_opts[CURLOPT_CONNECTTIMEOUT] = max($conn_timeout/1000,1);
		}
```




