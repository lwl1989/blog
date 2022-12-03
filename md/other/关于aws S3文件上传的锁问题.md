CreateTime:2017-02-17 13:35:58.0

#问题出现情况
由于客户端是（安卓、iOS、PC）多个版本的，因此，要传不同的文件到S3服务器，但是在上传完毕之后，出现了无法删除本地文件的提示  “text file busy”

#问题的分析
text file busy  即文件正忙，因此可能出现的原因有：

1.文件还没有上传完

2.文件被进程锁定了

3.文件资源没有被释放（因为aws的SDK是封装好的，因此当时我相信了他，么有考虑这一点，最后发现居然就是这一点）

#查看官方文档

    public Guzzle\Service\Resource\Model putObject( array $args = array() )

$arg是参数

    
```
$result = $client->putObject(array(
    'ACL' => 'string',
    'Body' => 'mixed type: string|resource|\Guzzle\Http\EntityBodyInterface',
    // Bucket is required
    'Bucket' => 'string',
    'CacheControl' => 'string',
    'ContentDisposition' => 'string',
    'ContentEncoding' => 'string',
    'ContentLanguage' => 'string',
    'ContentLength' => integer,
    'ContentType' => 'string',
    'Expires' => 'mixed type: string (date format)|int (unix timestamp)|\DateTime',
    'GrantFullControl' => 'string',
    'GrantRead' => 'string',
    'GrantReadACP' => 'string',
    'GrantWriteACP' => 'string',
    // Key is required
    'Key' => 'string',
    'Metadata' => array(
        // Associative array of custom 'string' key names
        'string' => 'string',
        // ... repeated
    ),
    'ServerSideEncryption' => 'string',
    'StorageClass' => 'string',
    'WebsiteRedirectLocation' => 'string',
    'SSECustomerAlgorithm' => 'string',
    'SSECustomerKey' => 'string',
    'SSECustomerKeyMD5' => 'string',
    'SSEKMSKeyId' => 'string',
    'RequestPayer' => 'string',
));
```

官方示例

```
$result = $client->putObject(array(
    'Bucket' => $bucket,
    'Key'    => 'data.txt',
    'Body'   => 'Hello!'
));
```

#强制获取文件返回方式（DEBUG过程）
首先，我们尝试用获取到返回值回调的方式进行判断（我们认为是文件还在上传过程中）。下面是官方的demo:

```
$result = $client->putObject(array(
    'Bucket'     => $bucket,
    'Key'        => 'data_from_file.txt',
    'SourceFile' => $pathToFile,
    'Metadata'   => array(
        'Foo' => 'abc',
        'Baz' => '123'
    )
));

// We can poll the object until it is accessible
$client->waitUntil('ObjectExists', array(
    'Bucket' => $this->bucket,
    'Key'    => 'data_from_file.txt'
));
```

然而问题并没有得到解决，但是文件很明显的是上传上去了的。同时，证明了putObject是个同步的操作而不是异步，因为他产生了很明显的延迟。

#继续查阅文档

```
$client->putObject(array(
    'Bucket' => $bucket,
    'Key'    => 'data_from_stream.txt',
    'Body'   => fopen($pathToFile, 'r+')
));
```

发现S3上传是可以通过资源或者字符流方式进行上传，因此，修改代码。

```
    try{
			if(isset($data['SourceFile'])) {
				$data['Body'] = fopen($data['SourceFile'],'r+');
				unset($data['SourceFile']);
			}
			$this->_s3->putObject($data);
			fclose($data['Body']);
		} catch(\Exception $e) {
			return $e->getMessage();
	}
```

不再相信S3会自动释放文件资源，果然，本地文件已被删除。

#推测

AWS的S3的SDK是没有对文件进行释放，而它内部普遍采用static的方法和类，因此文件释放的时机是由PHP决定。
所以出现了text file busy的错误。

#解决方案

不要相信SDK能处理好一切，对于本地的资源类型和需要上锁的操作交由自己来管理。